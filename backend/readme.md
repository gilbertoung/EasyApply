# easyApply — Robô de Candidaturas Inteligente
> README do Desenvolvedor · Frontend (Next.js) · Backend (FastAPI) · IA · Infra

---

## Time

| Membro | Função | Responsabilidades |
|--------|--------|-------------------|
| **ANDRE** | Backend Lead | Arquitetura, banco, FastAPI, JWT, Stripe, scraper Vagas.com, painel admin |
| **GILBERTO** | Backend + Infra | Docker, repositório, CRUD perfil, scraper Indeed/Catho, middleware de cotas, deploy Railway, testes E2E |
| **HEBERSON** | IA + Integrações | Claude API, upload PDF, fila RQ, SendGrid, planos, e-mails, deploy Vercel, analytics |
| **JADER** | Frontend Lead | Next.js, login/cadastro, perfil, dashboard, pricing, onboarding, landing page |

---

## Índice

1. [Visão Geral](#1-visão-geral)
2. [Stack Tecnológica](#2-stack-tecnológica)
3. [Estrutura do Repositório](#3-estrutura-do-repositório)
4. [Como Rodar Localmente](#4-como-rodar-localmente)
5. [Banco de Dados](#5-banco-de-dados)
6. [Backend — Endpoints da API](#6-backend--endpoints-da-api)
7. [Frontend — Páginas e Rotas](#7-frontend--páginas-e-rotas)
8. [IA de Matching — Claude API](#8-ia-de-matching--claude-api)
9. [Fila Assíncrona — Redis Queue](#9-fila-assíncrona--redis-queue)
10. [Pagamentos — Stripe](#10-pagamentos--stripe)
11. [Controle de Limites por Plano](#11-controle-de-limites-por-plano)
12. [Deploy em Produção](#12-deploy-em-produção)
13. [Links Úteis](#13-links-úteis)

---

## 1. Visão Geral

O **easyApply** é um SaaS de candidaturas automáticas a vagas de emprego. O usuário cadastra seu currículo e preferências, e o robô busca vagas em múltiplas plataformas, avalia a compatibilidade usando IA (Claude API) e aplica automaticamente nas vagas com maior score.

### Fluxo principal

```
Usuário cadastra perfil + currículo PDF
        ↓
Scraper busca vagas (Indeed, Vagas.com, Catho)
        ↓
Claude API compara currículo × vaga → score 0-100
        ↓
Score ≥ mínimo do usuário?
   ├── SIM → Robô aplica + envia e-mail de confirmação
   └── NÃO → Vaga ignorada, log registrado
```

### Planos do SaaS

| Plano | Preço | Aplicações/mês | Vagas/dia | Plataformas |
|-------|-------|---------------|-----------|-------------|
| **Free** | Grátis | 10 | 20 | Indeed |
| **Pro** | R$ 29,90/mês | 100 | 100 | Indeed + Vagas.com |
| **Premium** | R$ 59,90/mês | Ilimitado | Ilimitado | Todas |

---

## 2. Stack Tecnológica

| Camada | Tecnologia | Responsável |
|--------|-----------|-------------|
| Frontend | Next.js 14 + TypeScript + Tailwind + Shadcn/ui | JADER |
| Backend | Python + FastAPI + SQLAlchemy + Alembic | ANDRE |
| Banco de dados | PostgreSQL 15 + Redis 7 | ANDRE / GILBERTO |
| Storage | Supabase Storage | HEBERSON |
| Robô (scraping) | Playwright Python + fake-useragent | GILBERTO |
| IA de matching | Claude API — Anthropic SDK Python | HEBERSON |
| Fila assíncrona | Redis Queue (RQ) | HEBERSON |
| Pagamentos | Stripe — Subscriptions + Webhooks | ANDRE |
| Notificações | SendGrid — Templates dinâmicos | HEBERSON |
| Deploy frontend | Vercel — CI/CD via GitHub | HEBERSON |
| Deploy backend | Railway — PostgreSQL + Redis | GILBERTO |

---

## 3. Estrutura do Repositório

Monorepo com frontend e backend separados na raiz.

```
/
├── frontend/                  ← projeto Next.js
│   ├── app/                   ← rotas (App Router)
│   ├── components/            ← componentes reutilizáveis
│   │   ├── ui/                ← componentes base (Shadcn)
│   │   ├── dashboard/         ← tabela de candidaturas, banner de plano
│   │   ├── forms/             ← formulários de perfil e upload
│   │   ├── onboarding/        ← barra de progresso e etapas
│   │   └── pricing/           ← cards de plano
│   ├── lib/                   ← utilitários e configurações
│   ├── hooks/                 ← custom hooks
│   └── types/                 ← tipos TypeScript
│
├── backend/                   ← projeto FastAPI
│   ├── routers/               ← endpoints por domínio
│   │   ├── auth.py
│   │   ├── users.py
│   │   ├── curriculos.py
│   │   ├── aplicacoes.py
│   │   ├── stripe.py
│   │   └── admin.py
│   ├── models/                ← modelos SQLAlchemy
│   ├── schemas/               ← schemas Pydantic
│   ├── services/              ← regras de negócio
│   │   ├── matching.py        ← integração Claude API
│   │   └── scraper/           ← indeed.py, vagas_com.py, catho.py
│   ├── jobs/                  ← jobs da fila RQ
│   │   ├── matching.py
│   │   ├── aplicacao.py
│   │   └── notificacao.py
│   ├── core/                  ← config, db, segurança, middleware
│   ├── alembic/               ← migrations do banco
│   └── main.py
│
├── docker-compose.yml
├── .env.example
└── README.md
```

### Convenções de branch

| Branch | Uso |
|--------|-----|
| `main` | Produção — protegida. PR obrigatório com ao menos 1 aprovação |
| `develop` | Integração — base para todas as features |
| `feature/nome` | Nova funcionalidade. Ex: `feature/auth-jwt` |
| `hotfix/nome` | Correção urgente em produção |

### Padrão de commits — Conventional Commits

```
feat: adiciona endpoint POST /auth/register
fix: corrige bug no parser do PDF
chore: atualiza dependências do requirements.txt
docs: atualiza README com endpoints do Stripe
refactor: extrai lógica de matching para service
test: adiciona testes Pytest para /auth/login
```

---

## 4. Como Rodar Localmente

### Pré-requisitos

- **Node.js 18+** — para o frontend
- **Python 3.11+** — para o backend
- **Docker Desktop** — para PostgreSQL e Redis
- **Git** — controle de versão

### Passo a passo

**1. Clone o repositório**

```bash
git clone https://github.com/easyapply/easyapply.git
cd easyapply
```

**2. Suba o banco e o Redis com Docker**

```bash
docker compose up -d
```

Serviços iniciados:
- PostgreSQL 15 → `localhost:5432`
- Redis 7 → `localhost:6379`
- pgAdmin (opcional) → `localhost:5050`

**3. Configure as variáveis de ambiente**

```bash
cp .env.example .env
# Edite o .env com suas chaves (ver seção abaixo)
```

**4. Instale e rode o backend**

```bash
cd backend
pip install -r requirements.txt
alembic upgrade head        # roda as migrations
uvicorn main:app --reload
```

- API disponível em: `http://localhost:8000`
- Swagger (docs interativos) em: `http://localhost:8000/docs`
- ReDoc em: `http://localhost:8000/redoc`

**5. Instale e rode o frontend** (em outro terminal)

```bash
cd frontend
npm install
npm run dev
```

- App disponível em: `http://localhost:3000`

### Variáveis de ambiente

Todas as variáveis ficam no arquivo `.env` na raiz do backend. **Nunca suba o `.env` para o repositório.**

| Variável | Serviço | Quem usa |
|----------|---------|----------|
| `DATABASE_URL` | PostgreSQL | Backend |
| `REDIS_URL` | Redis | Backend |
| `SECRET_KEY` | JWT (gerado aleatoriamente) | Backend |
| `SUPABASE_URL` | Supabase Storage | Backend |
| `SUPABASE_KEY` | Supabase Storage | Backend |
| `ANTHROPIC_API_KEY` | Claude API | Backend |
| `STRIPE_SECRET_KEY` | Stripe | Backend |
| `STRIPE_WEBHOOK_SECRET` | Stripe Webhooks | Backend |
| `SENDGRID_API_KEY` | SendGrid | Backend |
| `NEXT_PUBLIC_API_URL` | URL da API | Frontend |
| `NEXT_PUBLIC_STRIPE_KEY` | Stripe (chave pública) | Frontend |

> **Dica:** Para gerar um `SECRET_KEY` seguro rode: `python -c "import secrets; print(secrets.token_hex(32))"`

---

## 5. Banco de Dados

O banco principal é **PostgreSQL 15**. As migrations são gerenciadas pelo **Alembic**. Nunca edite o banco diretamente em produção — toda mudança de schema deve vir de uma migration versionada.

### Principais tabelas

| Tabela | Descrição | Campos principais |
|--------|-----------|-------------------|
| `users` | Usuários cadastrados | `id, nome, email, senha_hash, ativo, plano_id, is_admin` |
| `planos` | Planos Free/Pro/Premium | `id, nome, preco_mensal, limite_aplicacoes, limite_vagas_dia` |
| `assinaturas` | Assinaturas Stripe | `id, user_id, plano_id, stripe_subscription_id, status, validade` |
| `curriculos` | Currículos enviados | `id, user_id, s3_url, texto_extraido, criado_em` |
| `vagas` | Vagas coletadas pelos scrapers | `id, titulo, empresa, url, plataforma, descricao, salario, localidade` |
| `aplicacoes` | Candidaturas realizadas | `id, user_id, vaga_id, score_ia, status, aplicado_em` |
| `logs` | Log de execução dos jobs | `id, aplicacao_id, mensagem, nivel, criado_em` |

### Relacionamentos

```
users ──── assinaturas ──── planos
  │
  ├──── curriculos
  │
  └──── aplicacoes ──── vagas
              │
              └──── logs
```

### Comandos Alembic

```bash
# Criar nova migration após alterar um model
alembic revision --autogenerate -m "descricao da mudanca"

# Aplicar todas as migrations pendentes
alembic upgrade head

# Ver status atual das migrations
alembic current

# Voltar uma migration
alembic downgrade -1

# Ver histórico de migrations
alembic history
```

---

## 6. Backend — Endpoints da API

A documentação interativa completa fica em `http://localhost:8000/docs` após rodar o backend.

### 6.1 Autenticação — `/auth`

| Método | Rota | Descrição | Auth |
|--------|------|-----------|------|
| `POST` | `/auth/register` | Cria usuário. Associa ao plano Free automaticamente. | ❌ |
| `POST` | `/auth/login` | Retorna `access_token` (30min) e `refresh_token` (7 dias). | ❌ |
| `POST` | `/auth/refresh` | Renova o `access_token` usando o `refresh_token`. | ❌ |
| `POST` | `/auth/verify-email` | Valida token de confirmação de e-mail (válido por 24h). | ❌ |
| `POST` | `/auth/reset-password` | Redefine senha via token enviado por e-mail (válido por 1h). | ❌ |

### 6.2 Usuário — `/users`

| Método | Rota | Descrição | Auth |
|--------|------|-----------|------|
| `GET` | `/users/me` | Retorna perfil completo + currículo ativo + plano atual + uso. | ✅ |
| `PUT` | `/users/me` | Atualiza preferências e dados do perfil. | ✅ |
| `DELETE` | `/users/me` | Soft delete — define `ativo=false`. Não apaga do banco. | ✅ |

### 6.3 Currículo — `/curriculos`

| Método | Rota | Descrição | Auth |
|--------|------|-----------|------|
| `POST` | `/curriculos/upload` | Recebe PDF (max 5MB), salva no Supabase, extrai texto com pdfplumber. | ✅ |
| `GET` | `/curriculos/me` | Retorna currículo ativo do usuário com preview do texto. | ✅ |

### 6.4 Aplicações — `/aplicacoes`

| Método | Rota | Descrição | Auth |
|--------|------|-----------|------|
| `GET` | `/aplicacoes` | Lista candidaturas. Filtros: `status`, `plataforma`, `page`, `limit`. | ✅ |
| `GET` | `/aplicacoes/{id}` | Detalhes de uma candidatura: vaga, score, pontos fortes, lacunas. | ✅ |
| `GET` | `/jobs/status` | Status atual da fila RQ: jobs pendentes, em execução e falhos. | ✅ |

### 6.5 Stripe — `/stripe`

| Método | Rota | Descrição | Auth |
|--------|------|-----------|------|
| `POST` | `/stripe/checkout-session` | Cria sessão de checkout para o plano escolhido. Retorna URL. | ✅ |
| `POST` | `/stripe/webhook` | Recebe eventos do Stripe. Verificado via assinatura, não por JWT. | ❌ |
| `GET` | `/stripe/portal` | Redireciona para o portal de gestão da assinatura no Stripe. | ✅ |

### 6.6 Admin — `/admin`

> Requer campo `is_admin=true` no banco. Middleware próprio separado do JWT comum.

| Método | Rota | Descrição |
|--------|------|-----------|
| `GET` | `/admin/users` | Lista todos os usuários com filtros por plano e status. |
| `GET` | `/admin/metrics` | MRR, conversão Free→Pro/Premium, distribuição de planos, total de aplicações. |
| `PUT` | `/admin/users/{id}/bloquear` | Bloqueia ou desbloqueia uma conta (toggle `ativo`). |
| `GET` | `/admin/logs` | Log de erros do robô e da fila nas últimas 24h. |

---

## 7. Frontend — Páginas e Rotas

| Rota | Descrição | Login |
|------|-----------|-------|
| `/` | Landing page do produto | ❌ |
| `/pricing` | Comparação de planos e preços | ❌ |
| `/register` | Criação de conta | ❌ |
| `/login` | Acesso à conta | ❌ |
| `/forgot-password` | Recuperação de senha | ❌ |
| `/onboarding/perfil` | Etapa 1 — dados pessoais e preferências | ✅ |
| `/onboarding/curriculo` | Etapa 2 — upload do currículo PDF | ✅ |
| `/onboarding/configurar` | Etapa 3 — score mínimo e palavras-chave | ✅ |
| `/onboarding/plano` | Etapa 4 — escolha do plano (Free já selecionado) | ✅ |
| `/onboarding/concluido` | Etapa 5 — confirmação e próximos passos | ✅ |
| `/dashboard` | Tabela de candidaturas com filtros e paginação | ✅ |
| `/analytics` | Métricas de uso: score médio, taxa de aprovação, gráfico semanal | ✅ |
| `/profile` | Edição de perfil e preferências | ✅ |
| `/privacidade` | Política de privacidade (LGPD) | ❌ |
| `/termos` | Termos de uso | ❌ |
| `/admin` | Painel administrativo interno | Admin |

### Componentes principais

| Componente | Localização | Descrição |
|-----------|-------------|-----------|
| `ScoreBadge` | `components/ui/` | Badge colorido por faixa: verde ≥80, amarelo ≥60, vermelho <60 |
| `PlanBanner` | `components/dashboard/` | Banner com uso atual vs limite do plano e link de upgrade |
| `ApplicationTable` | `components/dashboard/` | Tabela de candidaturas com filtros, paginação e ordenação |
| `OnboardingProgress` | `components/onboarding/` | Barra de progresso das 5 etapas |
| `PlanCard` | `components/pricing/` | Card de plano com benefícios e botão de assinatura |
| `UploadCurriculo` | `components/forms/` | Dropzone para PDF com preview do texto extraído |

### Regras de autenticação no frontend

```
Rota pública  → qualquer usuário pode acessar
Rota privada  → redireciona para /login se não autenticado
Rota admin    → redireciona para /dashboard se não for admin
Pós-cadastro  → sempre redireciona para /onboarding/perfil
Pós-login     → redireciona para /dashboard
```

---

## 8. IA de Matching — Claude API

O matching é o **diferencial central do produto**. A IA compara o texto extraído do currículo com a descrição da vaga e retorna um score estruturado.

### Função principal

```python
# backend/services/matching.py

def match_vaga(curriculo_texto: str, descricao_vaga: str) -> dict:
    """
    Envia currículo + vaga para a Claude API.
    Retorna dict com score, pontos_fortes, lacunas e recomendacao.
    Em caso de erro no parse do JSON, retorna score=0.
    """
```

### Estrutura do retorno

```json
{
  "score": 85,
  "pontos_fortes": ["Python", "FastAPI", "experiência com APIs REST"],
  "lacunas": ["Kubernetes", "experiência em fintech"],
  "recomendacao": "Perfil compatível. Destacar experiência em APIs REST na carta de apresentação."
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `score` | `int` (0-100) | Percentual de compatibilidade |
| `pontos_fortes` | `list[str]` | Habilidades que batem com os requisitos da vaga |
| `lacunas` | `list[str]` | Requisitos ausentes no currículo |
| `recomendacao` | `str` | Sugestão personalizada para o candidato |

### Atenção ao parsear o retorno

> ⚠️ A Claude API pode retornar texto antes do JSON. **Sempre use `try/except`** ao fazer `json.loads()` e implemente fallback para `score=0` em caso de retorno inválido. Nunca confie que o retorno será JSON puro sem validação.

---

## 9. Fila Assíncrona — Redis Queue

As candidaturas não acontecem em tempo real. Elas são processadas em background pela fila **RQ (Redis Queue)**, garantindo escalabilidade e reprocessamento em caso de falha.

### Jobs e ordem de execução

```
Scraper encontra vaga elegível
        ↓
[1] score_matching_job(user_id, vaga_id)
        ↓
    Score ≥ mínimo do usuário?
    ├── NÃO → encerra, log registrado
    └── SIM ↓
[2] aplicar_vaga_job(user_id, vaga_id)
        ↓
    Limite do plano atingido?
    ├── SIM → encerra, e-mail de upgrade enviado
    └── NÃO ↓
[3] notificar_usuario_job(user_id, aplicacao_id)
        ↓
    E-mail SendGrid enviado ao usuário
```

### Arquivos dos jobs

| Job | Arquivo | Biblioteca |
|-----|---------|-----------|
| `score_matching_job` | `jobs/matching.py` | anthropic SDK |
| `aplicar_vaga_job` | `jobs/aplicacao.py` | Playwright |
| `notificar_usuario_job` | `jobs/notificacao.py` | SendGrid SDK |

### Regras da fila

- Retry automático: **máximo 3 tentativas** com backoff exponencial
- Se todas as tentativas falharem: status da aplicação vai para `falhou` e log é registrado
- Verificação de limite do plano acontece **dentro** do `aplicar_vaga_job`, não antes
- `GET /jobs/status` retorna jobs pendentes, em execução e falhos das últimas 24h

---

## 10. Pagamentos — Stripe

O Stripe gerencia assinaturas recorrentes. **Nunca armazene dados de cartão** — o Stripe cuida de toda a parte sensível.

### Fluxo de assinatura

```
1. Usuário escolhe plano em /pricing
2. Frontend chama POST /stripe/checkout-session
3. Backend cria sessão → retorna URL do checkout
4. Usuário é redirecionado para o checkout do Stripe
5. Stripe processa → dispara webhook POST /stripe/webhook
6. Backend atualiza assinaturas + plano_id do usuário
7. Usuário é redirecionado para /dashboard com plano ativo
```

### Eventos do webhook

| Evento | Ação no backend |
|--------|----------------|
| `checkout.session.completed` | Ativar assinatura — atualizar `plano_id` do usuário |
| `invoice.payment_succeeded` | Renovação OK — registrar nova `validade` |
| `invoice.payment_failed` | Notificar usuário por e-mail, bloquear acesso temporariamente |
| `customer.subscription.deleted` | Cancelamento — rebaixar para plano Free |

### Testar localmente com Stripe CLI

```bash
# Instalar Stripe CLI
# https://docs.stripe.com/stripe-cli

# Escutar e redirecionar webhooks para o backend local
stripe listen --forward-to localhost:8000/stripe/webhook

# Simular eventos manualmente
stripe trigger checkout.session.completed
stripe trigger customer.subscription.deleted
```

> **Cartão de teste:** `4242 4242 4242 4242` · qualquer data futura · qualquer CVV

---

## 11. Controle de Limites por Plano

O middleware `check_plan_limit` verifica o plano do usuário antes de qualquer candidatura ser enfileirada. Os contadores ficam no Redis com TTL automático.

### Resposta quando o limite é atingido

```
HTTP 429 Too Many Requests

{
  "detail": "Limite de aplicações do plano Free atingido.",
  "upgrade_url": "/pricing"
}
```

### Chaves no Redis

| Chave | Valor | TTL |
|-------|-------|-----|
| `aplicacoes:{user_id}:{mes}` | Contador de aplicações do mês | 30 dias |
| `vagas_dia:{user_id}:{data}` | Vagas buscadas no dia | 24 horas |
| `refresh_token:{user_id}` | Refresh token JWT | 7 dias |
| `verify_token:{token}` | Token de confirmação de e-mail | 24 horas |
| `reset_token:{token}` | Token de recuperação de senha | 1 hora |

---

## 12. Deploy em Produção

| Componente | Plataforma | Responsável |
|-----------|-----------|-------------|
| Frontend Next.js | Vercel | HEBERSON |
| Backend FastAPI | Railway | GILBERTO |
| PostgreSQL | Railway | GILBERTO |
| Redis | Upstash ou Railway | GILBERTO |
| Storage PDF | Supabase | HEBERSON |
| Domínio + SSL | Registro.br / Vercel / Railway | GILBERTO |

### Deploy automático

- **Frontend:** push em `main` → Vercel faz deploy automático
- **Backend:** push em `main` → GitHub Actions roda testes → Railway faz deploy

### CI/CD — GitHub Actions

```
Push em develop → roda testes Pytest + testes E2E Playwright
Pull Request em main → obrigatório passar nos testes antes do merge
Push em main → deploy automático no Railway e Vercel
```

> ⚠️ **Nunca suba o arquivo `.env` para o repositório.** Configure todas as variáveis diretamente no painel da Vercel e do Railway. O arquivo `.env.example` no repositório serve apenas como referência.

---

## 13. Links Úteis

### Frontend

| Recurso | URL |
|---------|-----|
| Next.js 14 Docs | https://nextjs.org/docs |
| Next.js App Router | https://nextjs.org/docs/app |
| Shadcn/ui | https://ui.shadcn.com |
| React Hook Form | https://react-hook-form.com |
| Zod | https://zod.dev |
| Recharts | https://recharts.org/en-US |

### Backend

| Recurso | URL |
|---------|-----|
| FastAPI Docs | https://fastapi.tiangolo.com |
| FastAPI Security (JWT) | https://fastapi.tiangolo.com/tutorial/security |
| SQLAlchemy | https://docs.sqlalchemy.org |
| Alembic Tutorial | https://alembic.sqlalchemy.org/en/latest/tutorial.html |
| Pydantic | https://docs.pydantic.dev |
| Pytest | https://docs.pytest.org |

### IA e Integrações

| Recurso | URL |
|---------|-----|
| Claude API Docs | https://docs.anthropic.com |
| Anthropic Python SDK | https://github.com/anthropics/anthropic-sdk-python |
| Prompt Engineering Guide | https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview |
| pdfplumber | https://github.com/jsvine/pdfplumber |
| SendGrid Python SDK | https://github.com/sendgrid/sendgrid-python |
| SendGrid Templates | https://docs.sendgrid.com/ui/sending-email/how-to-send-an-email-with-dynamic-templates |

### Infra e Deploy

| Recurso | URL |
|---------|-----|
| Stripe Subscriptions | https://docs.stripe.com/billing/subscriptions/overview |
| Stripe Webhooks | https://docs.stripe.com/webhooks |
| Stripe Testing | https://docs.stripe.com/testing |
| Playwright Python | https://playwright.dev/python/docs/intro |
| RQ (Redis Queue) | https://python-rq.org |
| Supabase Storage | https://supabase.com/docs/guides/storage |
| Railway Docs | https://docs.railway.app |
| Vercel Docs | https://vercel.com/docs |
| Docker Compose | https://docs.docker.com/compose |
| GitHub Actions | https://docs.github.com/en/actions |

### Padrões e Legal

| Recurso | URL |
|---------|-----|
| Conventional Commits | https://www.conventionalcommits.org/pt-br |
| dbdiagram.io | https://dbdiagram.io |
| LGPD — Lei 13.709/2018 | https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm |

---

*easyApply · Versão 1.0 · Abril 2026*
