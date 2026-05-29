# LP Seguro Garantia (ASAS)

Guia rapido para integracao da Landing Page (LP) com o fluxo de contratacao.

## 1) O que a LP faz

A LP executa este fluxo:

1. Usuario preenche formulario (nome, CNPJ, email, telefone).
2. LP autentica na API ASAS (OAuth).
3. LP consulta limite de credito do tomador (`consultaTomador`).
4. LP exibe resultado:
- Sem limite: mostra mensagem de nao aprovado.
- Com limite: mostra estado aprovado e botao `Contratar`.
5. No clique em `Contratar`, LP dispara redirecionamento hibrido:
- Mesmo dominio do destino: salva dados no `localStorage` e navega para contratacao.
- Dominio diferente: envia payload em `?d=` na URL de destino.

## 2) Variaveis que o cliente pode ajustar

No script da LP, ajuste estes valores:

- `ASAS_API_BASE`
- `ASAS_OAUTH_URL`
- `ASAS_CONSULTA_URL`
- `ASAS_BASIC_AUTH`
- `ASAS_TIMEOUT`
- `ASAS_REDIRECT_PROD_BASE_URL`
- `ASAS_REDIRECT_HML_BASE_URL`
- `ASAS_MODALIDADE_ID`

Observacao:

- Ambiente HML e ativado com `?hml` na URL da LP.
- Sem `?hml`, a LP usa destino de producao.
- Em HML, a LP exibe um badge flutuante no rodape informando a URL de disparo.

## 3) Como a consulta e feita

### 3.1 OAuth (token)

A LP chama:

- `POST {ASAS_OAUTH_URL}`
- `Content-Type: application/x-www-form-urlencoded`
- Header `Authorization: Basic ...`

Body enviado (URLSearchParams):

```txt
grant_type=password
username=integracao.bbmnet@asasinsurance.com
password={G5QL!(Avsrf
```

### 3.2 Consulta de credito

A LP chama:

- `POST {ASAS_CONSULTA_URL}`
- `Authorization: Bearer <access_token>`
- `Content-Type: application/json`

Payload enviado:

```json
{
  "telefone": "11999999999",
  "cnpjConsulta": "12345678000199",
  "email": "cliente@empresa.com",
  "idCorretor": 245,
  "name": "Nome da Empresa"
}
```

## 4) Como a resposta e exibida

A LP analisa:

- `msgs[0].businessMessage`
- `limites[0].limiteCredito`

Regras:

- Se `businessMessage` for diferente de `Sucesso.` (ou `Success.`), mostra erro.
- Se `limiteCredito <= 0`, mostra estado sem limite.
- Se `limiteCredito > 0`, mostra estado aprovado com botao `Contratar`.

## 5) Dados de redirecionamento gerados pela LP

Quando aprovado, a LP monta este objeto:

```json
{
  "telefone": "11999999999",
  "cnpj": "12345678000199",
  "email": "cliente@empresa.com",
  "limiteCredito": 150000,
  "idCorretor": 245,
  "idTomador": 987654,
  "idModalidade": 17,
  "termoAceite": true
}
```

## 6) Modo hibrido de redirect

### 6.1 Mesmo dominio (same-origin)

A LP salva no `localStorage`:

- `consultaCredito` (objeto acima em JSON)
- `asasAccessToken` (token OAuth)

Depois navega para:

- `ASAS_REDIRECT_BASE_URL`

### 6.2 Dominio diferente (cross-origin)

A LP envia:

- `?d=<payload_base64_urlencoded>`

Exemplo de URL:

```txt
https://dominio-cliente.com/editais/contratacao/?d=eyJ0ZWxlZm9uZSI6Ii4uLiJ9
```

## 7) O que o cliente deve receber para dar sequencia

No lado cliente (destino), precisa existir suporte para pelo menos um destes caminhos:

1. Ler `localStorage.consultaCredito` + `localStorage.asasAccessToken` (quando same-origin).
2. Ler `?d=`, decodificar payload e hidratar `localStorage` antes de abrir contratacao (quando cross-origin).

Se o cliente nao tratar nenhum dos dois, o usuario chega na URL mas a contratacao nao inicia no estado esperado.

## 8) Checklist rapido

- Ajustou URLs de PROD/HML.
- Testou LP com `?hml`.
- Validou consulta aprovada e sem limite.
- Validou redirect same-origin (localStorage).
- Validou redirect cross-origin (`?d=`).
- Validou que o destino consome os dados para continuar a contratacao.
