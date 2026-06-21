# Agents — Fase 1: Esqueleto ponta-a-ponta

> Definição dos agentes, contratos de dados, estado interno e interface de execução
> da Fase 1. A Fase 1 é **totalmente determinística** — nenhuma chamada a LLM.
> Os agentes descritos aqui são os mesmos da arquitetura final; o que muda na Fase 2
> é a substituição do stand-in pelo agente de IA real.

---

## Execução local

### Pré-requisitos

| Ferramenta | Versão mínima | Verificar |
|---|---|---|
| Python | 3.12 | `python3 --version` |
| uv | qualquer | `uv --version` |
| Node | 20 | `node --version` |
| npm | 10 | `npm --version` |
| git | qualquer | `git --version` |

Instalar `uv` se não tiver: `curl -LsSf https://astral.sh/uv/install.sh | sh`

### Backend

```bash
cd backend

# criar e ativar ambiente virtual
uv venv
source .venv/bin/activate       # macOS/Linux
# .venv\Scripts\activate        # Windows

# instalar dependências (produção + dev)
uv pip install -e ".[dev]"

# rodar o servidor
uvicorn archscan.api.main:app --reload --port 8000
```

Verificar: `curl http://localhost:8000/health` → `{"status":"ok"}`

### Frontend

```bash
cd frontend

npm install
npm run dev
```

Verificar: abrir `http://localhost:5173`

### Via Docker (alternativa)

```bash
# sobe backend + frontend juntos
docker compose up

# rebuild após mudanças de dependência
docker compose up --build
```

### Testes

```bash
cd backend
source .venv/bin/activate

pytest                        # todos os testes
pytest tests/steps/           # só uma pasta
pytest -v                     # verbose
```

### Pipeline CLI (disponível após Fase 1 implementada)

```bash
cd backend
source .venv/bin/activate

# analisar repos remotos
python -m archscan run \
  --repos https://github.com/org/repo-a https://github.com/org/repo-b \
  --flow-name "checkout" \
  --output grafo.json

# analisar repos locais (útil para desenvolvimento)
python -m archscan run \
  --repos /caminho/local/repo-a /caminho/local/repo-b \
  --output grafo.json

# usando arquivo de lista
python -m archscan run \
  --repos-file repos.txt \
  --output grafo.json
```

`repos.txt` — uma URL ou caminho local por linha; linhas em branco e `#` são ignorados.

Inspecionar o resultado: `cat grafo.json | python -m json.tool`

---

## Os dois agentes

| Agente | Fase 1 | Fase 2 |
|---|---|---|
| **Orquestrador** | script Python que percorre as 7 etapas em ordem | mesmo papel, mas com pontos de decisão de IA |
| **Analisador** | stand-in determinístico: converte fatos → nós/ligações | agente de IA com ferramentas de leitura de código |

---

## Orquestrador

### Papel

Conduz o fluxo das 7 etapas na ordem fixa. Chama cada ferramenta determinística,
acumula o estado entre etapas e ao final produz o grafo JSON.

Na Fase 1 não há pontos de decisão de IA: o orquestrador é um script linear que
executa as etapas e propaga falhas como avisos (nunca interrompe o pipeline todo).

### Estado interno

O orquestrador carrega um `RunContext` que cresce a cada etapa:

```python
@dataclass
class RunContext:
    repos: list[str]                    # entrada do usuário
    flow_name: str | None               # nome opcional do fluxo
    workdir: Path                       # diretório temporário de trabalho

    # populado em cada etapa:
    manifesto: list[RepoEntry]          # após Etapa 1
    stacks: list[RepoStack]             # após Etapa 2
    fatos: dict[str, list[Fato]]        # após Etapa 3 — chave: repo
    analises: list[AnaliseResult]       # após Etapa 4
    grafo: Architecture                 # após Etapa 5
    grafo_validado: Architecture        # após Etapa 6
    # Etapa 7 serializa grafo_validado para JSON
```

### Ciclo de vida

```
start(repos, flow_name)
  │
  ├─ etapa 1: collect(repos) → manifesto
  ├─ etapa 2: detect_stack(manifesto) → stacks
  ├─ etapa 3: discovery(stacks) → fatos          # paralelo por repo
  ├─ etapa 4: analisar(fatos) → analises         # paralelo por repo (stand-in na Fase 1)
  ├─ etapa 5: build(analises) → grafo
  ├─ etapa 6: validate(grafo) → grafo_validado
  └─ etapa 7: render(grafo_validado) → JSON
```

**Falha em etapa.** Se um repo falha no clone ou no discovery, o orquestrador registra
o erro em `RunContext.erros`, continua com os repos restantes e entrega o grafo
parcial ao final. O pipeline nunca aborta por falha de um repo.

### Como executar

```bash
# via CLI (Fase 1)
python -m archscan run \
  --repos https://github.com/org/repo-a https://github.com/org/repo-b \
  --flow-name "checkout" \
  --output grafo.json

# ou com arquivo de repos
python -m archscan run \
  --repos-file repos.txt \
  --output grafo.json
```

`repos.txt` — uma URL ou caminho local por linha, linhas em branco e `#` ignorados.

```bash
# via API (Fase 3 em diante, mas o endpoint já existe desde a Fase 1 para testes)
curl -X POST http://localhost:8000/analyze \
  -H "Content-Type: application/json" \
  -d '{"repos": ["https://github.com/org/repo-a"], "flow_name": "checkout"}'
```

---

## Analisador (stand-in — Fase 1)

### Papel

Converte a lista de `Fato` de um codebase em `AnaliseResult` (nós + ligações +
papel). Na Fase 1 não há interpretação: cada fato vira diretamente um nó e uma
ligação, sem leitura extra do código.

É **1 instância por codebase**, rodam em **paralelo** (pool limitado).

### Como estar (comportamento)

```
recebe: fatos: list[Fato], repo: RepoEntry

para cada fato:
  1. garante que existe um Node com id = f"{fato.tipo}:{fato.nome}"
     - escopo = "interno" se o nome for resolvido, "horizonte" se for externo
  2. garante que existe um Node para o próprio serviço (id = f"service:{repo.repo}")
  3. cria uma Edge ligando serviço → recurso (ou recurso → serviço, pela direção)

devolve: AnaliseResult com nos, ligacoes, papel
```

**Regra de horizonte no stand-in.** Na Fase 1, todo recurso cuja URL/nome contém
`.` (ex.: `api.acme.com`) ou não aparece em nenhum outro repo é marcado como
`escopo = "horizonte"` automaticamente. A Fase 2 refina isso com julgamento de IA.

### Entrada e saída

```python
# entrada
@dataclass
class EntradaAnalisador:
    repo: RepoEntry
    fatos: list[Fato]

# saída
@dataclass
class AnaliseResult:
    repo: str               # nome do repo
    nos: list[Node]
    ligacoes: list[Edge]
    papel: str              # descrição do papel do codebase — na Fase 1: derivado dos fatos
```

---

## Contratos de dados

Estes são os tipos estáveis do projeto — definidos em `backend/archscan/model/`.
**Não mudam entre fases**; o que muda é como são preenchidos (stand-in vs. IA).

### Fato

Saída do discovery. Unidade mínima de evidência.

```python
class Fato(BaseModel):
    tipo:      Literal["topic", "queue", "api", "bucket", "database"]
    direcao:   Literal["produz", "consome", "chama", "lê", "escreve"]
    nome:      str                   # nome do recurso ("customer.created", "api.acme.com")
    detalhe:   list[str] | None      # rotas, colunas, etc.
    evidencia: str                   # "caminho/relativo.py:42 | trecho de código"
    confianca: Literal["alta", "baixa"]
```

### Node

Nó do grafo. Um recurso ou serviço.

```python
class Node(BaseModel):
    id:         str                  # ID canônico: "tipo:nome" (ex.: "topic:customer.created")
    tipo:       Literal["topic", "queue", "api", "bucket", "database", "service", "horizon"]
    nome:       str
    tecnologia: str | None           # "kafka", "sqs", "postgres", "s3", etc.
    escopo:     Literal["interno", "horizonte"]
    evidencias: list[str]            # lista de "arquivo:linha | trecho"
    avisos:     list[str] = []
```

### Edge

Ligação direcional entre dois nós.

```python
class Edge(BaseModel):
    id:        str                   # "origem_id→destino_id"
    origem:    str                   # Node.id
    destino:   str                   # Node.id
    direcao:   Literal["produz", "consome", "chama", "lê", "escreve"]
    detalhe:   list[str] | None      # ex.: rotas ["/charge", "/refund"]
    evidencias: list[str]
```

### RepoEntry

Uma entrada do manifesto (saída da Etapa 1).

```python
class RepoEntry(BaseModel):
    repo:       str                  # nome curto (último segmento da URL)
    url:        str                  # URL original ou caminho local
    path_local: Path                 # onde foi clonado
    sha:        str                  # commit HEAD exato
    branch:     str
    status:     Literal["ok", "error"]
    erro:       str | None = None
```

### RepoStack

Manifesto enriquecido com stacks (saída da Etapa 2).

```python
class RepoStack(BaseModel):
    repo:       str
    path_local: Path
    sha:        str
    branch:     str
    stacks:     list[Literal["java", "python", "node", "dotnet", "go", "terraform", "generic"]]
```

### Architecture

O grafo completo do fluxo. Saída final do pipeline (Etapa 7).

```python
class Architecture(BaseModel):
    flow_name:        str | None
    nos:              list[Node]
    ligacoes:         list[Edge]
    repos_analisados: list[RepoEntry]
    profundidade:     Literal["baixo", "médio", "alto"] = "baixo"
    avisos:           list[str]      # avisos não-bloqueantes
    erros:            list[str]      # erros registrados (repos com falha, etc.)
```

---

## O que a Fase 2 substitui

| Item | Fase 1 (stand-in) | Fase 2 (IA) |
|---|---|---|
| Analisador | fato → nó/ligação direto | agente interpreta, usa ferramentas de leitura |
| Horizonte | heurística (`.` no nome) | julgamento do agente sobre o contexto |
| Papel do codebase | derivado do tipo de fatos | descrição em linguagem natural |
| Fato de baixa confiança | mantido com aviso | agente decide promover ou descartar |
| Ambiguidade no Montar | separa com aviso | IA decide com reasoning |
| Loop no Validar | não existe | IA corrige erros duros em loop limitado |

---

*Derivado de ARCHSCAN.md — Fase 1.*
