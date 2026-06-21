# Spec — Fase 0: Estrutura base do projeto

> Spec SDD da Fase 0. Derivado do ARCHSCAN.md (seção "Fase 0 — Estrutura base do projeto").
> Status: **planejado**

---

## O quê

Criar o esqueleto completo de diretórios e configurações do monorepo Archscan,
com backend (Python + FastAPI) e frontend (React + React Flow) subindo vazios.

Nenhuma lógica de negócio é implementada aqui — só a estrutura que sustenta as
fases seguintes.

## Por quê

Sem uma organização estabelecida, cada fase acaba improvisando onde colocar as
coisas. A estrutura da Fase 0 espelha diretamente o design do ARCHSCAN.md:
`model/` é o contrato, `steps/` + `discovery/` + `agents/` são as 7 etapas,
`api/` expõe o backend. Errar a organização agora é barato; errar nas Fases 2–3
tem custo alto.

## Não é (non-goals)

- Nenhuma lógica de discovery, análise, orquestração ou renderização.
- Nenhuma integração com LLM/API.
- Nenhuma UI de visualização ou grafo.
- Testes de funcionalidade (não há funcionalidade ainda).

---

## Estrutura de diretórios alvo

```
archscan/
├── README.md
├── ARCHSCAN.md
├── docker-compose.yml
├── docs/
│   └── specs/
│       └── fase-0-estrutura-base.md   ← este arquivo
│
├── backend/
│   ├── pyproject.toml
│   ├── Dockerfile
│   └── archscan/
│       ├── __init__.py
│       ├── model/                     # Fato, Node, Edge, Architecture (contrato)
│       │   └── __init__.py
│       ├── orchestrator/              # roteiro das 7 etapas
│       │   └── __init__.py
│       ├── steps/                     # etapas determinísticas
│       │   ├── __init__.py
│       │   ├── collect.py             # 1 Coletar
│       │   ├── detect_stack.py        # 2 Detect stack
│       │   ├── build.py               # 5 Montar
│       │   ├── validate.py            # 6 Validar
│       │   └── render.py              # 7 Renderizar
│       ├── discovery/                 # 3 Discovery — uma ferramenta por linguagem
│       │   ├── __init__.py
│       │   ├── base.py                # contrato do Fato + interface comum
│       │   └── generic.py             # discovery-generic (fallback)
│       ├── agents/                    # 4 Analisar — interface Agent + adaptadores LLM
│       │   └── __init__.py
│       └── api/                       # FastAPI: /analyze, /query, /health
│           ├── __init__.py
│           └── main.py
│   └── tests/
│       └── __init__.py
│
└── frontend/
    ├── package.json
    ├── tsconfig.json
    ├── vite.config.ts
    ├── index.html
    ├── Dockerfile
    └── src/
        ├── main.tsx
        ├── App.tsx
        ├── pages/                     # tela de input, tela de resultado
        ├── components/                # visualizador (React Flow), nós, painéis
        ├── icons/                     # ícones por tipo/tecnologia
        └── api/                       # cliente da API do backend
```

---

## Stack e dependências

### Backend

**Runtime:** Python 3.12+

**Dependências principais** (`pyproject.toml`):
- `fastapi` — framework da API
- `uvicorn[standard]` — servidor ASGI
- `pydantic` — validação e modelo de dados (base para `model/`)
- `gitpython` — operações git (clone, sha) na Fase 1
- `httpx` — cliente HTTP assíncrono (chamadas externas futuras)
- `anthropic` — SDK para os agentes de IA (Fase 2)

**Dev:**
- `pytest`, `pytest-asyncio` — testes
- `ruff` — linting e formatação

### Frontend

**Runtime:** Node 20+

**Dependências principais** (`package.json`):
- `react`, `react-dom` — framework UI
- `@xyflow/react` — React Flow (visualizador de grafo)
- `typescript` — tipagem
- `vite` — build tool

**Dev:**
- `@types/react`, `@types/react-dom`
- `eslint`

### Infraestrutura

**`docker-compose.yml`:**
- `backend` — imagem Python, monta `./backend`, roda uvicorn, porta `8000`
- `frontend` — imagem Node, monta `./frontend`, roda vite dev, porta `5173`

---

## Critérios de aceite

1. **Estrutura completa** — todos os diretórios e arquivos placeholder existem conforme o diagrama acima.
2. **Backend sobe** — `docker compose up backend` (ou `uvicorn archscan.api.main:app`) retorna `200` em `GET /health`.
3. **Frontend sobe** — `docker compose up frontend` (ou `vite dev`) serve a página em `http://localhost:5173`.
4. **`docker compose up`** sobe ambos sem erro.
5. **Módulos importáveis** — `python -c "import archscan"` não levanta `ImportError`.
6. **Lint passa** — `ruff check backend/` sem erros.
7. **Placeholders documentados** — cada módulo stub tem uma docstring de 1 linha indicando o que será implementado na fase correta.

---

## Plano de implementação

As tarefas abaixo são a sequência de implementação da Fase 0.
Ordem importa onde há dependência; onde não há, podem ser paralelizadas.

### T0.1 — Configuração do backend

- Criar `backend/pyproject.toml` com deps e configuração do ruff/pytest.
- Criar `backend/archscan/__init__.py` e os `__init__.py` de cada subpacote.
- Criar `backend/archscan/api/main.py` com `GET /health` retornando `{"status": "ok"}`.
- Criar `backend/Dockerfile` (imagem `python:3.12-slim`, instala deps, expõe porta 8000).

### T0.2 — Placeholders do backend

Criar os arquivos stub de cada módulo (só docstring + `pass` ou `raise NotImplementedError`):

- `model/__init__.py` — "Modelos de dados: Fato, Node, Edge, Architecture. Implementado na Fase 1."
- `orchestrator/__init__.py` — "Orquestrador das 7 etapas. Implementado na Fase 1."
- `steps/collect.py` — "Etapa 1: clona repositórios. Implementado na Fase 1."
- `steps/detect_stack.py` — "Etapa 2: detecta a stack de cada repo. Implementado na Fase 1."
- `steps/build.py` — "Etapa 5: monta o grafo único. Implementado na Fase 1."
- `steps/validate.py` — "Etapa 6: valida o grafo. Implementado na Fase 1."
- `steps/render.py` — "Etapa 7: serializa o grafo em JSON. Implementado na Fase 1."
- `discovery/base.py` — "Contrato do Fato e interface de discovery. Implementado na Fase 1."
- `discovery/generic.py` — "discovery-generic (fallback). Implementado na Fase 1."
- `agents/__init__.py` — "Interface Agent agnóstica de vendor. Implementado na Fase 2."

### T0.3 — Configuração do frontend

- Inicializar projeto Vite + React + TypeScript em `frontend/`.
- Instalar `@xyflow/react`.
- Criar `src/App.tsx` com placeholder: título "Archscan" e mensagem "em construção".
- Criar pastas `pages/`, `components/`, `icons/`, `api/` com `index.ts` vazio em cada.
- Criar `frontend/Dockerfile` (imagem `node:20-slim`, instala deps, expõe porta 5173).

### T0.4 — docker-compose e wiring

- Criar `docker-compose.yml` com os dois serviços (`backend`, `frontend`).
- Mapear volumes para hot-reload em dev.
- Verificar `docker compose up` sobe ambos sem erro.

### T0.5 — Validação dos critérios de aceite

- Rodar `curl http://localhost:8000/health` → `{"status": "ok"}`.
- Abrir `http://localhost:5173` → página carrega.
- Rodar `python -c "import archscan"` sem erro.
- Rodar `ruff check backend/` sem erros.

---

## Dependências entre fases

Esta fase não depende de nenhuma outra. Ela é o pré-requisito de todas as demais:

```
Fase 0 (estrutura) → Fase 1 (esqueleto lógico) → Fase 2 (IA) → …
```

---

*Spec derivado do ARCHSCAN.md — Documento de Design.*
