# Shopee Afiliados — Automação n8n

Automação de publicação de ofertas da Shopee em um canal do Telegram, executada de
forma contínua com **n8n** em uma instância própria. Este repositório guarda a
**versão versionada e sanitizada** do workflow de produção, além da documentação
necessária para reproduzi-lo em outro ambiente.

> **Este repositório não contém credenciais.** Todos os segredos foram substituídos
> por placeholders (`SEU_APP_ID_AQUI`, `SEU_APP_SECRET_AQUI`, `SEU_CHAT_ID_AQUI`).
> Veja [Proteção de dados e credenciais](#proteção-de-dados-e-credenciais).

---

## Sumário

- [Objetivo](#objetivo)
- [Fluxo geral](#fluxo-geral)
- [Tecnologias](#tecnologias)
- [Arquitetura de hospedagem](#arquitetura-de-hospedagem)
- [Organização do projeto](#organização-do-projeto)
- [Requisitos](#requisitos)
- [Configuração](#configuração)
- [Backup e restauração](#backup-e-restauração)
- [Execução local](#execução-local)
- [Proteção de dados e credenciais](#proteção-de-dados-e-credenciais)

---

## Objetivo

A automação monitora a **API de Afiliados da Shopee** por ofertas de acessórios de
eletrônicos e publica automaticamente as promoções encontradas em um canal do
Telegram, respeitando regras de deduplicação para não repetir o mesmo produto em
excesso. Quando qualquer etapa falha, um alerta de erro é enviado ao mesmo canal.

Resumo do que a automação faz:

1. Consulta a API de afiliados da Shopee por dezenas de palavras-chave de produtos.
2. Filtra os resultados para manter apenas categorias de eletrônicos.
3. Remove ofertas já publicadas recentemente.
4. Publica até 3 ofertas por rodada (a cada 30 minutos) com imagem, preço,
   desconto, loja e link.
5. Envia alerta ao Telegram caso o workflow apresente erro.

---

## Fluxo geral

O workflow publicado neste repositório tem **11 nós** e duas cadeias: a de
publicação e a de alerta de erros.

### Cadência

| Gatilho | Configuração |
| --- | --- |
| `A cada 30 minutos` (Schedule Trigger) | Expressão cron `0 */30 * * * *` — executa a cada 30 minutos |

### Cadência de publicação

```
A cada 30 minutos
        │
        ▼
PREP 1 — Montar request Shopee          (Code)
        │  monta lista de palavras-chave, monta a query GraphQL
        │  productOfferV2 e o texto a ser assinado
        ▼
Crypto                                  (Crypto — SHA256)
        │  calcula a assinatura da requisição
        ▼
PREP 3 — productOfferV2 (eletrônicos)   (HTTP Request)
        │  POST na API GraphQL de afiliados da Shopee
        ▼
PREP 4 — Extrair ofertas                (Code)
        │  achata a resposta (aliases k0, k1, ...), intercala os
        │  grupos (round-robin) e escapa HTML para o Telegram
        ▼
PREP 5 — Filtrar eletrônicos            (Code)
        │  lista branca de termos técnicos (usb, hdmi, carregador,
        │  fone, mouse, teclado...) + lista de bloqueio
        ▼
PREP 6 — Deduplicar (itemId)            (Code)
        │  usa o static data do workflow para evitar repetições:
        │  no máximo 3 por rodada e no máximo 2x por dia por item
        ▼
PREP 7 — Publicar oferta (Telegram)     (Telegram — sendPhoto)
           envia imagem + legenda com nome, preço, desconto,
           loja, link e hashtags
```

### Cadência de alerta de erros

```
Error Trigger
        │
        ▼
Montar alerta                           (Code)
        │  monta a mensagem com horário (America/Sao_Paulo),
        │  nome do workflow, nó responsável e detalhes do erro
        ▼
Enviar alerta (Telegram)                (Telegram — sendMessage)
```

> A cadeia de alerta é acionada pelo campo **Error Workflow** nas configurações do
> próprio workflow (autoreferência).

### Regras de negócio relevantes

- **Assinatura das requisições:** a API exige um cabeçalho `Authorization` no formato
  `SHA256 Credential=<appId>, Timestamp=<timestamp>, Signature=<assinatura>`, onde a
  assinatura é o hash SHA256 de `appId + timestamp + payload + appSecret`.
- **Limite da API:** o máximo é 50 resultados por palavra-chave; o nó PREP 1 usa
  *aliases* GraphQL (`k0`, `k1`, ...) para consultar vários nichos em uma única chamada.
- **Filtro de categoria:** somente acessórios de eletrônicos passam (cabo, adaptador,
  fone, carregador, mouse, teclado, webcam, smartwatch, peças para Arduino/ESP32 etc.).
- **Nunca inventa preço:** os valores publicados são apenas os retornados pela API.
- **Estado de deduplicação:** fica no *static data* do workflow (dentro do n8n), não
  neste repositório.

---

## Tecnologias

| Tecnologia | Uso no projeto |
| --- | --- |
| **n8n** | Motor de automação que executa o workflow |
| **n8n nodes** | `Schedule Trigger`, `Code`, `Crypto`, `HTTP Request`, `Telegram`, `Error Trigger` |
| **API de Afiliados da Shopee** | Endpoint GraphQL público de ofertas (`productOfferV2`) |
| **Telegram Bot API** | Publicação das ofertas e envio de alertas (via credencial nativa do n8n) |
| **Docker / Docker Compose** | Execução do n8n na VM Oracle Cloud |
| **Oracle Cloud (OCI)** | Infraestrutura da VM que hospeda o n8n |

---

## Arquitetura de hospedagem

```
┌─────────────────────────────────────────────┐
│  VM Oracle Cloud                      │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │  Docker Compose                       │  │
│  │                                       │  │
│  │  ┌─────────┐      ┌────────────────┐  │  │
│  │  │  n8n    │◄────►│  Volume de     │  │  │
│  │  │         │      │  dados (SQLite │  │  │
│  │  └────┬────┘      │  + config)     │  │  │
│  │       │           └────────────────┘  │  │
│  └───────┼───────────────────────────────┘  │
│          │ HTTPS                            │
└──────────┼──────────────────────────────────┘
           │
    ┌──────┴───────┐        ┌──────────────────┐
    │  Reverse     │        │  Serviços        │
    │  proxy +     │        │  externos:       │
    │  domínio     │        │  • API Shopee    │
    │  dinâmico    │        │  • Telegram API  │
    └──────────────┘        └──────────────────┘
```

- O **n8n** roda em contêiner (Docker/Docker Compose) dentro de uma VM Oracle Cloud.
- O estado (workflows, credenciais, execuções, *static data* de deduplicação) vive em
  um **volume persistente** apontado pelo Docker Compose — sem ele, a automação perde
  histórico e credenciais.
- O acesso à interface e à API do n8n é feito por **HTTPS** atrás de um proxy reverso
  com domínio dinâmico, exposto apenas com TLS.
- As credenciais (Telegram, etc.) são armazenadas **criptografadas pelo próprio n8n**,
  com uma chave de criptografia que permanece apenas na VM.

> Endereços, IPs, domínios e identificadores da infraestrutura de produção são
> deliberadamente omitidos deste repositório.

---

## Organização do projeto

```
shopee-afiliados-n8n/
├── README.md                                  # este documento
├── .gitignore                                 # impede a commitação de segredos
├── .env.example                               # exemplo de variáveis de ambiente
└── workflows/
    └── shopee-afiliados-producao.json         # workflow de produção (sanitizado)
```

- `workflows/shopee-afiliados-producao.json` é um **export fiel** do workflow em
  produção, com os seguintes ajustes de segurança:
  - `appId` → `SEU_APP_ID_AQUI`
  - `appSecret` → `SEU_APP_SECRET_AQUI`
  - `chatId` do Telegram → `SEU_CHAT_ID_AQUI`
  - removidos: dados de compartilhamento, e-mails, identificadores internos da
    instância, histórico de deduplicação e versões internas.

---

## Requisitos

Para reproduzir o projeto em outro ambiente:

| Requisito | Detalhes |
| --- | --- |
| n8n | Instância própria (self-hosted), versão recente com suporte aos nós `Code` (v2), `HTTP Request` (v4.5), `Schedule Trigger` (v1.2) e `Telegram` (v1.2) |
| Docker + Docker Compose | Para subir o n8n conforme a arquitetura de produção |
| Conta de afiliado da Shopee | Com `appId` e `appSecret` gerados no painel de afiliados |
| Bot do Telegram | Criado via `@BotFather`, com token e um chat/canal onde o bot seja administrador |
| Domínio + TLS | Para expor a interface do n8n com HTTPS |

---

## Configuração

### 1. Variáveis de ambiente

Copie o exemplo e preencha com **seus** valores (o `.env` é ignorado pelo Git):

```bash
cp .env.example .env
```

| Variável | Descrição | Valor exemplo |
| --- | --- | --- |
| `SHOPEE_APP_ID` | ID da aplicação no painel de afiliados | `SEU_APP_ID_AQUI` |
| `SHOPEE_APP_SECRET` | Segredo da aplicação (usado na assinatura SHA256) | `SEU_APP_SECRET_AQUI` |
| `TELEGRAM_CHAT_ID` | ID do chat/canal de destino | `SEU_CHAT_ID_AQUI` |

> O workflow publicado usa **placeholders literais** no lugar destes valores, de
> modo que o arquivo não funcione sozinho ao ser importado. Após importar, substitua
> os placeholders pelos valores reais **diretamente na interface do n8n** — nunca
> commite os valores preenchidos.

### 2. Credenciais direto no n8n

Prefira o cofre de credenciais do n8n a qualquer arquivo:

1. No n8n: **Credentials → Add credential → Telegram API**.
2. Cole o **token do bot** e salve. Dê um nome como `Telegram account`.
3. Nos nós `PREP 7 - Publicar oferta (Telegram)` e `Enviar alerta (Telegram)`,
   selecione essa credencial em **Credential for Telegram API**.

Recomendações:

- O **token do bot nunca** deve aparecer em nós `Code`, em variáveis de ambiente
  versionadas ou em exportações de workflow.
- Para o `appSecret` da Shopee, o caminho mais seguro é movê-lo para um mecanismo
  seguro do n8n (por exemplo, um *credential* do tipo que se adeque ao seu ambiente)
  em vez de mantê-lo como texto dentro do nó `Code`. Enquanto isso não for feito,
  o valor real deve existir apenas na instância de produção.

### 3. Pós-importação

Após importar `workflows/shopee-afiliados-producao.json`:

1. Preencha os placeholders (`SEU_APP_ID_AQUI`, `SEU_APP_SECRET_AQUI`, `SEU_CHAT_ID_AQUI`).
2. Selecione a credencial do Telegram nos dois nós de envio.
3. Reaponte o campo **Error Workflow** nas configurações do workflow para o próprio
   workflow (o valor do export aponta para o ID da instância de produção original).
4. Confirme o fuso horário e o *timeout* de execução nas configurações, se necessário.
5. Ative o workflow manualmente e monitore a primeira execução.

---

## Backup e restauração

### Backup

O que precisa ser preservado:

| Item | Onde fica | Como proteger |
| --- | --- | --- |
| Workflows | Banco de dados do n8n | Export pela UI (baixa lateral) ou pela API |
| Credenciais | Banco de dados do n8n (criptografadas) | Backup do volume + **chave de criptografia** do n8n |
| Chave de criptografia | Arquivo de configuração do n8n | Armazenar fora do Git, em cofre seguro |
| Volume do Docker | VM Oracle Cloud | Snapshot do volume / backup do diretório apontado no Compose |

Procedimento recomendado:

1. **Snapshot do volume** do n8n na VM antes de qualquer manutenção.
2. **Export dos workflows** pela UI do n8n (não versione o arquivo cru — ele contém
   placeholders apenas neste repositório; o export da produção contém valores reais).
3. Guardar o export e a chave de criptografia **fora** do repositório (cofre/segredo).

### Restauração

1. Subir a instância com Docker Compose, apontando o volume para o backup.
2. Restaurar o arquivo de configuração com a **mesma chave de criptografia** — sem
   ela, as credenciais armazenadas não podem ser decriptografadas.
3. Importar os exports de workflow (ou restaurar o banco de dados inteiro).
4. Verificar gatilhos, credenciais e **Error Workflow** antes de ativar.
5. Rodar uma execução manual e conferir o canal de teste antes de liberar em produção.

> Nunca restaure um backup sobre a instância de produção sem antes confirmar que o
> snapshot atual foi salvo.

---

## Execução local

Para rodar uma cópia de teste sem tocar em produção:

```bash
docker run -d \
  --name n8n-local \
  -p 5678:5678 \
  -e N8N_HOST=localhost \
  -e N8N_PORT=5678 \
  -e N8N_PROTOCOL=http \
  -e GENERIC_TIMEZONE=America/Sao_Paulo \
  -e TZ=America/Sao_Paulo \
  -v n8n_local_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

1. Abra `http://localhost:5678` e crie o usuário inicial.
2. Importe `workflows/shopee-afiliados-producao.json`.
3. Preencha os placeholders e cadastre a credencial do Telegram.
4. Use um chat de **teste** (não o canal de produção) para validar os envios.
5. Ative o workflow apenas quando estiver satisfeito com os resultados.

---

## Proteção de dados e credenciais

Medidas adotadas neste repositório:

- **Nenhum segredo é versionado:** `appSecret`, token do bot, IDs de chat e chaves de
  API estão fora do Git. O workflow publicado usa placeholders claramente identificados.
- **`.gitignore` robusto:** ignora `.env*` (exceto `.env.example`), chaves privadas,
  certificados, bancos de dados, logs, volumes do n8n e exportações brutas
  (`exports/brutos/`) que ainda não foram revisadas.
- **Sem dados operacionais:** identificadores internos da instância, e-mails,
  histórico de deduplicação e domínios/IPs da infraestrutura foram removidos do export.
- **Repositório privado** no GitHub — não torne público sem decisão explícita do
  mantenedor.
- **Produção intocada:** este repositório é um espelho documentado; nenhuma alteração
  aqui é aplicada automaticamente à instância em produção.

Boas práticas recomendadas ao mantenedor:

1. Nunca commite um export feito diretamente da produção sem antes sanitizá-lo
   (procure por `appSecret`, `appId`, `chatId`, `bot<id>:<token>`).
2. Prefira armazenar segredos em credenciais do n8n a texto dentro de nós `Code`.
3. Rode com frequência um scanner de segredos (ex.: `gitleaks`, `trufflehog`) antes
   de cada push.
4. Se um segredo for acidentalmente commitado, **remover no commit seguinte não
   basta** — o conteúdo permanece no histórico. Nesse caso, revogue/rotacione a
   chave e limpe o histórico (ex.: `git filter-repo`) antes de qualquer push.
5. Mantenha o token da API do n8n e a chave de criptografia em um cofre de segredos,
   nunca no repositório nem no terminal compartilhado.

---

## Publicação no GitHub

O repositório está preparado para ser publicado em um repositório **privado** chamado
`shopee-afiliados-n8n`. Procedimento recomendado (o token nunca deve ser digitado em
scripts, histórico do terminal ou neste arquivo):

```bash
# 1. Instalar o GitHub CLI (uma vez)
winget install --id GitHub.cli

# 2. Autenticar (fluxo interativo, aberto por você)
gh auth login

# 3. Confirmar a conta autenticada
gh auth status

# 4. Publicar (a partir da pasta do projeto)
cd shopee-afiliados-n8n
gh repo create shopee-afiliados-n8n --private --source=. --remote=origin --push
```

Antes de qualquer `push`, rode uma varredura de segredos:

```bash
git ls-files | xargs grep -nE "bot[0-9]{6,}:|[A-Z0-9]{25,}|-----BEGIN|sk-[A-Za-z0-9]{20,}|gh[pousr]_[A-Za-z0-9]{20,}"
```

Regras:

- O repositório deve permanecer **privado**. Torná-lo público exige decisão explícita
  do mantenedor.
- Nunca faça `push` de um export cru da produção: sanitize antes (procure por
  `appSecret`, `appId`, `chatId` e `bot<id>:<token>`).
- Se um segredo já tiver sido commitado, removê-lo no próximo commit **não basta** —
  revogue/rotacione a chave e limpe o histórico antes de qualquer `push`.

---

## Licença e uso

Projeto pessoal de automação, mantido como backup, documentação e portfólio.
Consulte o mantenedor antes de reutilizar o código.
