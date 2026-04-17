# Automação n8n — Processamento de e-mails com anexo CSV financeiro

---

## Objetivo do fluxo
Quando chegar um e-mail com anexo CSV:
1. identificar o anexo CSV;
2. salvar o arquivo original em disco;
3. ler e transformar os registros;
4. validar colunas mínimas;
5. consultar a cotação USD→BRL na Open ER-API;
6. calcular o valor convertido em BRL;
7. gerar um resumo consolidado;
8. responder ao remetente com o status do processamento.

A cotação é obtida na API pública **Open ER-API**, usando explicitamente `BRL` e `USD`. 

---

## Nodes utilizados
- **Email Trigger (IMAP)**
- **Set**
- **Code**
- **IF**
- **Save File on Disk**
- **Extract From File** 
- **HTTP Request**
- **Send Email**


---

## Estrutura do workflow

### 1) Trigger de e-mail

Campos importantes:
- caixa de entrada monitorada;
- marcar ou não como lido;
- baixar anexos;
- permitir binários.

---

### 2) Set Config

```json
{
  "storageType": "local",
  "localFolder": "/tmp/onfly_financeiro",
  "requiredColumns": ["data", "descricao", "categoria", "centro_custo", "valor_usd"],
  "replyFromName": "financeiro Onfly",
  "replySubjectPrefix": "Processamento CSV Financeiro",
  "cotacaoUrl": "https://open.er-api.com/v6/latest/USD"
}
```

---

### 3) Code — Identificar anexo CSV e Dados CSV
Responsabilidades:
- 
- ler os binários do e-mail;
- localizar anexo `.csv`;
- capturar remetente e assunto;
- preparar `status = no_csv` quando não houver CSV.

### Exemplo de lógica
```javascript
const item = $input.first();
const binary = item.binary || {};
const json = item.json || {};

let csvBinaryKey = null;
let csvFileName = null;

for (const [key, file] of Object.entries(binary)) {
  const fileName = (file.fileName || '').toLowerCase();
  const mimeType = (file.mimeType || '').toLowerCase();
  if (fileName.endsWith('.csv') || mimeType.includes('csv')) {
    csvBinaryKey = key;
    csvFileName = file.fileName || 'anexo.csv';
    break;
  }
}

const fromRaw =
  json.from?.value?.[0]?.address ||
  json.from?.text ||
  json.from ||
  json.sender ||
  '';

return [{
  json: {
    ...json,
    hasCsv: !!csvBinaryKey,
    csvBinaryKey,
    csvFileName,
    senderEmail: fromRaw,
    senderName: json.from?.value?.[0]?.name || '',
  },
  binary
}];
```

---

### 4) IF — Tem CSV?
#### Se **não**
Enviar resposta ao remetente informando:
- que nenhum anexo CSV foi encontrado;
- que o processamento não foi executado.

#### Se **sim**
Seguir para gravação do arquivo original.

---

### 5) Read/Write Files from Disk — Salvar CSV original
Use a operação **Write File to Disk**.
- **Input Binary Field**: expressão apontando para o anexo CSV encontrado.
- **File Path and Name**:
```text
={{ $json.localFolder || "/data/financeiro/entrada" }}/{{ $now.format("yyyyLLdd_HHmmss") }}_{{ $json.csvFileName }}
```

Esse node grava o binário no disco da máquina onde o n8n está rodando.

---

### 6) Extract From File — Ler CSV
Configure para:
- **Operation**: Extract From CSV
- **Input Binary Field**: o mesmo campo binário do CSV

Esse node converte o CSV para JSON para processamento dentro do próprio n8n

---

### 7) Code — Tratar e validar CSV
Responsabilidades:
- validar presença das colunas mínimas;
- remover espaços excedentes;
- tratar campos vazios;
- normalizar datas;
- converter `valor_usd` para número;
- separar linhas válidas e inválidas;
- registrar inconsistências.

#### Regras sugeridas
- aceitar datas como `DD/MM/YYYY`, `YYYY-MM-DD` e `DD-MM-YYYY`;
- transformar string vazia em `null`;
- remover múltiplos espaços internos em texto;
- aceitar valores com vírgula decimal;
- gerar motivo da invalidação por linha.

#### Exemplo de código
```javascript
const rows = $input.all().map(i => i.json);
const requiredColumns = ['data', 'descricao', 'categoria', 'centro_custo', 'valor_usd'];

function normalizeText(v) {
  if (v === undefined || v === null) return null;
  const s = String(v).replace(/\s+/g, ' ').trim();
  return s === '' ? null : s;
}

function normalizeDate(v) {
  const s = normalizeText(v);
  if (!s) return null;

  if (/^\d{4}-\d{2}-\d{2}$/.test(s)) return s;

  let m = s.match(/^(\d{2})\/(\d{2})\/(\d{4})$/);
  if (m) return `${m[3]}-${m[2]}-${m[1]}`;

  m = s.match(/^(\d{2})-(\d{2})-(\d{4})$/);
  if (m) return `${m[3]}-${m[2]}-${m[1]}`;

  return null;
}

function normalizeNumber(v) {
  if (v === undefined || v === null) return null;
  let s = String(v).trim();
  if (!s) return null;

  s = s.replace(/\s/g, '');

  if (s.includes(',') && s.includes('.')) {
    if (s.lastIndexOf(',') > s.lastIndexOf('.')) {
      s = s.replace(/\./g, '').replace(',', '.');
    } else {
      s = s.replace(/,/g, '');
    }
  } else if (s.includes(',')) {
    s = s.replace(',', '.');
  }

  const n = Number(s);
  return Number.isFinite(n) ? n : null;
}

const headerKeys = rows.length ? Object.keys(rows[0]) : [];
const missingColumns = requiredColumns.filter(c => !headerKeys.includes(c));

if (missingColumns.length) {
  return [{
    json: {
      status: 'invalid_structure',
      missingColumns,
      headerKeys,
      validRows: [],
      invalidRows: rows.map((r, idx) => ({
        rowNumber: idx + 2,
        reason: `Colunas obrigatórias ausentes: ${missingColumns.join(', ')}`,
        original: r
      }))
    }
  }];
}

const validRows = [];
const invalidRows = [];

rows.forEach((row, index) => {
  const line = {
    data: normalizeDate(row.data),
    descricao: normalizeText(row.descricao),
    categoria: normalizeText(row.categoria),
    centro_custo: normalizeText(row.centro_custo),
    valor_usd: normalizeNumber(row.valor_usd),
  };

  const reasons = [];
  if (!line.data) reasons.push('data inválida');
  if (!line.descricao) reasons.push('descricao vazia');
  if (!line.categoria) reasons.push('categoria vazia');
  if (!line.centro_custo) reasons.push('centro_custo vazio');
  if (line.valor_usd === null) reasons.push('valor_usd inválido');

  if (reasons.length) {
    invalidRows.push({
      rowNumber: index + 2,
      reason: reasons.join('; '),
      original: row,
      normalized: line
    });
  } else {
    validRows.push({
      rowNumber: index + 2,
      ...line
    });
  }
});

return [{
  json: {
    status: 'ok',
    headerKeys,
    missingColumns: [],
    totalRowsRead: rows.length,
    validCount: validRows.length,
    invalidCount: invalidRows.length,
    validRows,
    invalidRows
  }
}];
```

---

### 8) IF — Estrutura valida?
Se `status = invalid_structure`:
- enviar e-mail ao remetente informando as colunas ausentes;
- incluir as colunas encontradas no CSV;
- finalizar.

Se `status = ok`:
- seguir para a consulta da cotação.

---

### 9) HTTP Request — Cotação do dólar
Configuração:
- **Method**: GET
- **URL**: `https://open.er-api.com/v6/latest/USD`

No retorno, usar explicitamente:
- `rates.BRL`
- `rates.USD`

---

### 11) Code — Email Sucesso
Responsabilidades:
- aplicar taxa USD→BRL em cada linha válida;
- calcular totais;
- agrupar por categoria;
- listar inconsistências.
- gerar o resumo consolidado para email

#### Exemplo de código
```javascript
const finance = $('Tratar e validar CSV').first().json;
const meta = $('dados CSV').first().json;
const fx = $input.first().json;

const usdRate = Number(fx.rates?.USD || 1);
const brlRate = Number(fx.rates?.BRL || 0);

if (!Number.isFinite(brlRate) || brlRate <= 0) {
  throw new Error('Cotação USD→BRL inválida.');
}

const validRows = finance.validRows || [];
const invalidRows = finance.invalidRows || [];

const enrichedRows = validRows.map(r => {
  const valor_brl = Number((r.valor_usd * (brlRate / usdRate)).toFixed(2));
  return {
    ...r,
    cotacao_usd_brl: Number((brlRate / usdRate).toFixed(6)),
    valor_brl
  };
});

const totalUsd = enrichedRows.reduce((acc, r) => acc + r.valor_usd, 0);
const totalBrl = enrichedRows.reduce((acc, r) => acc + r.valor_brl, 0);

const byCategory = {};
for (const r of enrichedRows) {
  const key = r.categoria || 'SEM_CATEGORIA';
  if (!byCategory[key]) {
    byCategory[key] = { quantidade: 0, total_usd: 0, total_brl: 0 };
  }
  byCategory[key].quantidade += 1;
  byCategory[key].total_usd += r.valor_usd;
  byCategory[key].total_brl += r.valor_brl;
}

for (const key of Object.keys(byCategory)) {
  byCategory[key].total_usd = Number(byCategory[key].total_usd.toFixed(2));
  byCategory[key].total_brl = Number(byCategory[key].total_brl.toFixed(2));
}

const categoryLines = Object.entries(byCategory).map(([cat, v]) =>
  `- ${cat}: ${v.quantidade} registros | USD ${v.total_usd.toFixed(2)} | BRL ${v.total_brl.toFixed(2)}`
).join('\n');

const inconsistencies = invalidRows.length
  ? invalidRows.slice(0, 20).map(r => `- Linha ${r.rowNumber}: ${r.reason}`).join('\n')
  : '- Nenhuma inconsistência relevante';

return [{
  json: {
    to: meta.senderEmail,
    from: meta.replyFrom,
    subject: `[Financeiro] Resultado do processamento - ${meta.csvFileName}`,
    body: `Olá${meta.senderName ? ' ' + meta.senderName : ''},

Processamento concluído.

Arquivo processado: ${meta.csvFileName}
Registros lidos: ${finance.totalRowsRead}
Registros válidos: ${enrichedRows.length}
Registros inválidos/descartados: ${invalidRows.length}
Cotação do dólar (USD→BRL): ${(brlRate / usdRate).toFixed(6)}
Total em USD: ${totalUsd.toFixed(2)}
Total em BRL: ${totalBrl.toFixed(2)}

Resumo por categoria:
${categoryLines || '- Sem dados válidos'}

Inconsistências encontradas:
${inconsistencies}

Arquivo original salvo em:
${meta.savedFilePath}

Atenciosamente,
Automação Financeira`,
    summary: {
      cotacao_usd_brl: Number((brlRate / usdRate).toFixed(6)),
      totalRowsRead: finance.totalRowsRead,
      validCount: enrichedRows.length,
      invalidCount: invalidRows.length,
      totalUsd: Number(totalUsd.toFixed(2)),
      totalBrl: Number(totalBrl.toFixed(2)),
      byCategory,
      invalidRows
    }
  }
}];

---

### 13) Send Email — Responder ao remetente

Configuração:
- **To**: remetente do e-mail original
- **From**: configurável
- **Subject**:

---

## Fluxo resumido
```text
Email Trigger
  -> Set Config
  -> Code: Detect CSV
  -> IF: hasCsv?
      -> false -> Code: Build "sem anexo CSV" -> Send Email
      -> true
          -> Read/Write Files from Disk
          -> Extract From File (CSV)
          -> Code: Normalize and Validate CSV
          -> IF: status == invalid_structure?
              -> true -> Code: Build "estrutura inválida" -> Send Email
              -> false
                  -> HTTP Request (Open ER-API)
                  -> Code: Calculate + Summary
                  -> Code: Build Success Email
                  -> Send Email
```

---


## Comportamentos esperados

### Caso 1 — Sem anexo CSV
Resposta:
- informar que nenhum CSV foi encontrado;
- não falhar silenciosamente.

### Caso 2 — CSV com colunas faltantes
Resposta:
- informar colunas ausentes;
- informar colunas encontradas;
- não processar valores financeiros.

### Caso 3 — CSV válido com linhas inválidas
Resposta:
- processar somente as linhas válidas;
- informar quantidade descartada;
- detalhar inconsistências.

### Caso 4 — CSV totalmente válido
Resposta:
- informar totais;
- incluir cotação do dia;
- incluir agrupamento consolidado.

---

## Sugestão de CSV esperado

```csv
data,descricao,categoria,centro_custo,valor_usd
2026-04-16,Licença SaaS,Software,TI,120.50
16/04/2026,Passagem aérea,Viagem,Comercial,890.00
2026-04-16,Consultoria externa,Serviços,Financeiro,3500
```

---
