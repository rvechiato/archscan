# Archscan — Documento de Design

> Objetivo, princípios e o desenho do fluxo de ponta a ponta (as 7 etapas), mais
> frontend e visualizador. Documento vivo — enriquecido conforme avançamos.
>
> **Repositório:** https://github.com/rvechiato/archscan.git

---

## Objetivo

A partir de **uma lista de codebases que pertencem a um mesmo fluxo de negócio**,
produzir automaticamente o **mapa da arquitetura** daquele fluxo: quais são os
componentes (serviços, filas, tópicos, bancos, buckets) e como eles se integram.

Em uma frase: **você dá os repositórios, o Archscan devolve o mapa do fluxo.**

## Motivação

Um fluxo de negócio é implementado por vários codebases independentes. Entender
como tudo se conecta exige abrir repositório por repositório e montar o quadro na
cabeça — conhecimento que vive com poucas pessoas e desatualiza a cada mudança. O
Archscan tira esse mapa direto do código, então ele acompanha a realidade.

## Princípio central

**A IA lidera a análise, mas não inventa nada.** Um passe determinístico extrai os
fatos do código (cada um com `arquivo:linha`); a IA recebe esses fatos como
evidência e só pode afirmar uma integração se houver evidência que a prove. Isso
combina a riqueza da análise por IA com rastreabilidade — o mapa não mente.

---

## Desenho macro

```
  FRONTEND (entrada)
  o usuário cola URLs de repositórios ou sobe um .txt
  + nome do fluxo (opcional)  →  botão "Iniciar análise"
        │
        ▼  (o sistema monta um flow.yaml interno a partir dos repos)
  ┌─────────────────────────────────────────────────────────────┐
  │  ORQUESTRADOR (agente) — conduz o roteiro, decide na dúvida   │
  │                                                               │
  │   1) COLETAR      clona os repositórios                       │
  │        ▼                                                      │
  │   2) DETECT STACK identifica a linguagem de cada repo         │
  │        ▼                                                      │
  │   3) DISCOVERY    extrai os fatos (determinístico, por ling.) │
  │        ▼                                                      │
  │   4) ANALISAR     agentes de IA interpretam cada codebase     │
  │        ▼                                                      │
  │   5) MONTAR       junta num grafo; marca fronteiras (horizonte)│
  │        ▼                                                      │
  │   6) VALIDAR      confere evidência em tudo (loop de correção)│
  │        ▼                                                      │
  │   7) RENDERIZAR   exporta o grafo (JSON)                      │
  └─────────────────────────────────────────────────────────────┘
        │
        ▼
  VISUALIZADOR
  mapa navegável com ícones por nó; clicar mostra a evidência;
  busca em linguagem natural destaca componentes
```

O **input do usuário** é só a lista de repositórios (via frontend) — ele não
escreve YAML. O `flow.yaml` é uma **representação interna**, montada pelo sistema, e
serve para reproduzir a análise e (opcionalmente) refinar com `known_infra` /
`known_external`.

## Decisões já tomadas

- **Motor:** agentes de IA ancorados em evidência (nem IA pura, nem só detecção).
- **Agentes agnósticos de vendor:** definidos por prompt + schema + ferramentas,
  via chamada direta ao LLM — sem depender de nenhum assistente de código.
- **Fronteiras externas (horizonte):** integrações para codebases não fornecidos
  aparecem no mapa como ponto de fronteira, sem nada inferido, e expansíveis se o
  codebase for adicionado depois.
- **Início enxuto:** só o caminho essencial (as 7 etapas, de coletar a renderizar).
  Memória/busca, persistência e refinamentos ficam para depois.
- **Orquestração por agente:** um agente conduz o fluxo, acionando scripts
  determinísticos para o trabalho mecânico e decidindo nos pontos de ambiguidade
  (ver "Detalhamento do fluxo").

---

## Detalhamento do fluxo

### Modelo: agente que aciona ferramentas

O fluxo é conduzido por um **agente orquestrador**. Ele não faz o trabalho pesado
diretamente — **aciona ferramentas determinísticas** para as partes mecânicas,
**toma decisões** quando há ambiguidade, e **valida** o resultado antes de seguir.

| Faz parte | Quem faz | Por quê |
|---|---|---|
| Clonar, detectar stack, discovery, montar, validar, renderizar | **ferramentas** (determinístico) | precisa ser exato e reprodutível |
| Conduzir as etapas, decidir em ambiguidade, interpretar, descrever | **agentes** (IA) | exige julgamento |

O agente **segue um roteiro fixo** (as 7 etapas, nesta ordem) e só exerce liberdade
nos **pontos de decisão** — isso mantém o fluxo previsível apesar de ser conduzido
por IA.

### As 7 etapas (índice)

Cada etapa tem sua seção própria com o detalhe completo logo abaixo.

| # | Etapa | Natureza | O que faz |
|---|---|---|---|
| 1 | **Coletar** | determinístico | clona os repositórios |
| 2 | **Detect stack** | determinístico | identifica a(s) linguagem(ns) de cada repo |
| 3 | **Discovery** | determinístico | extrai os fatos (`arquivo:linha`), por linguagem |
| 4 | **Analisar** | **IA** | agentes interpretam cada codebase (com evidência) |
| 5 | **Montar** | det. + IA na ambiguidade | junta num grafo; casa integrações; marca horizontes |
| 6 | **Validar** | det. + IA no loop | confere as regras; loop de correção |
| 7 | **Renderizar** | determinístico | exporta o grafo (JSON) para o visualizador |

### Onde a IA decide (pontos de julgamento)

São os únicos pontos onde o agente exerce liberdade; o resto é determinístico:

- **Analisar** — interpretar fatos incertos, inferir papel e regras de negócio;
- **Montar** — ambiguidade de identidade (dois recursos parecidos são o mesmo?);
- **Validar** — raciocinar sobre erros duros e corrigir (loop limitado).

Os três usam **reasoning**, que escala com o seletor de profundidade (ver Etapa 4).

### Os agentes

**Orquestrador** — conduz o fluxo. *Ferramentas:* as etapas determinísticas
(coletar, detect stack, discovery, montar, validar, renderizar) + disparar os
analisadores. *Decide:* re-tentativas, ambiguidade de identidade, correção no loop.
*Saída:* o grafo final validado.

**Analisador** (1 por codebase) — interpreta um repositório. *Recebe:* os fatos do
discovery daquele repo. *Ferramentas:* leitura do código (ler arquivo, listar
pasta, buscar trecho) quando os fatos não bastam. *Devolve:* nós + ligações **com
evidência** + papel do codebase. *Regra:* nunca afirma integração sem evidência;
marca contraparte externa como candidata a horizonte.

---

## Etapa 1 — Coletar

Materializa localmente cada repositório da lista, para que as etapas seguintes
tenham o código em disco.

**Como faz.**
- Clone **shallow** (`--depth 1`) — só o estado atual, sem histórico, para
  velocidade.
- **Paralelo** — vários repos ao mesmo tempo, limitado por um pool.
- Aceita **URL remota** (GitHub etc.) e **caminho local** (útil para teste).

**Autenticação.** Assume-se que o utilizador já tem a **configuração local
resolvida** (credenciais git/SSH no ambiente). Suporte a tokens/GitHub App fica
para uma fase futura, se necessário.

**Entrada.** A lista de repos do `flow.yaml` interno (montado pelo frontend a
partir das URLs/arquivo).

**Saída.** Um **manifesto** com uma entrada por repo coletado:

```
{ repo, url, path_local, sha, branch }
```

O `sha` (commit atual) identifica a versão exata analisada.

**Falha de clone.** Se um repo falha (URL errada, sem permissão, grande demais), a
etapa re-tenta algumas vezes; persistindo a falha, **registra um alerta na área de
logs/erros** indicando que aquele codebase não pôde ser analisado, e **segue com os
demais** — o mapa sai parcial, com o aviso visível, em vez de travar tudo.

**Não faz.** Não olha o conteúdo do código (isso é discovery) nem detecta a
linguagem (isso é a etapa 2). Coletar só traz o código para o disco.

---

## Etapa 2 — Detect stack

Identifica a(s) linguagem(ns) e framework(s) de cada repositório. É o que permite o
orquestrador escolher qual ferramenta de discovery chamar na etapa seguinte.

**Como faz.** Procura **arquivos marcadores** que denunciam a stack — rápido e
determinístico:

| Marcador | Stack |
|---|---|
| `pom.xml`, `build.gradle` | Java |
| `requirements.txt`, `pyproject.toml` | Python |
| `package.json` | Node |
| `*.csproj` | .NET |
| `go.mod` | Go |
| `*.tf` | Terraform |

**Recursivo.** Procura na raiz **e em subpastas** — um repo real é frequentemente
`app/` (Java) + `infra/` (Terraform), ou um monorepo com serviços em linguagens
diferentes. Por isso um repo pode ter **várias stacks**.

**Entrada.** O manifesto da etapa 1.

**Saída.** O manifesto **enriquecido** com as stacks de cada repo:

```
{ repo, path_local, sha, branch, stacks: ["java", "terraform"] }
```

Essa lista de stacks é o que dispara o **multi-discovery** na etapa 3: para cada
stack, o orquestrador chama a ferramenta correspondente e junta os fatos.

**Quando não detecta nada.** O repo é marcado para o **`discovery-generic`** (o
fallback) — nunca fica sem análise — e registra um aviso na área de logs ("stack
não reconhecida, usando análise genérica").

**Não faz.** Não varre integrações (isso é discovery). Só identifica a tecnologia.

---

## Etapa 3 — Discovery

O **discovery** é a etapa que **descobre os fatos** de um codebase: os pontos onde
ele se conecta com o mundo. É a fundação da confiabilidade do Archscan — tudo que a
IA afirma depois precisa estar ancorado num fato que o discovery encontrou.

### Natureza: determinístico puro
O discovery **não usa IA**. É código que varre os arquivos e relata o que encontra
— **rápido, barato e reprodutível** (mesma entrada → mesmos fatos). A interpretação
(o que os fatos significam) é trabalho do agente analisador, na etapa seguinte.
Essa fronteira limpa é o que mantém o sistema confiável.

### Foco: pontos de integração que viram nós
O discovery não descreve o código todo — procura só **o que vira nó e ligação no
desenho**. Para cada codebase, responde:

- **Mensageria:** é **produtor** ou **consumidor**? De qual **tópico/fila** (nome)?
- **Chamadas de API:** para qual **host** chama? Em quais **rotas**?
- **Outros recursos:** quais buckets, bancos etc. acessa? (a evoluir)

### O que produz: o fato
A saída é uma lista de **fatos**, todos no mesmo formato. Este é o **contrato
central** do discovery — estável, venha o fato de qual ferramenta vier:

```
Fato = { tipo, direção, nome, detalhe?, evidência, confiança }

  tipo      → topic | queue | api | bucket | database | …
  direção   → produz | consome | chama | lê | escreve
  nome      → nome do recurso (tópico "customer.created", host "api.acme.com"…)
  detalhe?  → opcional (ex.: rotas de uma API: ["/charge", "/refund"])
  evidência → arquivo : linha + trecho de código
  confiança → alta (lib conhecida) | baixa (sinal genérico, lib não reconhecida)
```

A **direção** é o que permite casar produtor com consumidor e desenhar a seta no
sentido certo. A **evidência** é a prova. O discovery **só relata o que vê** — nunca
conclui nem nomeia componentes de negócio; isso é papel da IA.

### Organização: uma ferramenta por linguagem
A mesma integração tem **código diferente em cada linguagem** (`kafkaTemplate.send`
em Java, `producer.send` em Python…). Por isso o discovery é **uma ferramenta por
linguagem**, não um detector único que tenta dar conta de tudo:

```
orquestrador detecta a(s) stack(s) do codebase
   ├─ Java   → discovery-java
   ├─ Python → discovery-python
   ├─ Node   → discovery-node
   └─ (sem ferramenta dedicada) → discovery-generic   (fallback)
```

Cada ferramenta é **autocontida** e usa **a melhor técnica do seu ecossistema**
(`discovery-java` com parser de Java, `pom.xml`, `application.yml` do Spring;
`discovery-python` com `requirements.txt`, `.env`). A complexidade de *como*
analisar cada linguagem fica encapsulada — o orquestrador só sabe "Java → essa
ferramenta". E a regra de ouro: **ferramentas diferentes por dentro, contrato
(o fato) idêntico por fora**, então o passo `montar` recebe tudo no mesmo formato.

### Híbrido: dedicado + fallback
- **Ferramentas dedicadas** para as linguagens principais (Java, Python, Node…) —
  alta precisão onde está a maioria dos codebases.
- **`discovery-generic`** como fallback para stacks sem ferramenta dedicada —
  detecta sinais que independem de linguagem (URLs HTTP, `jdbc:...`, ARNs, strings
  que parecem tópico), sempre com `confiança: baixa`.

Assim, uma linguagem sem ferramenta dedicada **nunca vira um buraco total**.
Começamos só com a genérica e adicionamos dedicadas **sob demanda**.

### Duas camadas de confiança (dentro de cada ferramenta)
- **detectores específicos** — libs conhecidas (confiança **alta**);
- **detector genérico** — sinais soltos sem lib identificada (confiança **baixa**,
  marcado "lib não reconhecida").

Nunca se perde um ponto em silêncio: o que o específico não pega, o genérico
captura como incerto, e o agente decide depois. (Mesma filosofia do horizonte.)

### O que cada ferramenta identifica (detectores)

| Detector | Procura | Produz |
|---|---|---|
| **mensageria** | produção/consumo (`kafkaTemplate.send`, `@KafkaListener`, `@SqsListener`…) | fato `topic`/`queue` com **direção** e **nome** |
| **API / HTTP** | chamadas de saída (`@FeignClient`, `WebClient`, `axios`, `requests`, URLs) | fato `api` com **host** + **rotas** |
| **rotas expostas** | endpoints que o codebase oferece (`@RestController`, rotas) | o **papel** do codebase (o que recebe) |
| **config** | `application.yml`, `.env`, `*.tf` | resolve **nomes** vindos de variáveis |

(buckets e bancos entram conforme evoluímos — mesmo formato de fato.)

### Regras de comportamento
- **Ignora arquivos de teste** — mocks e fixtures contêm integrações falsas que
  poluiriam o mapa. Exclui por convenção de caminho/nome (`/test/`, `/tests/`,
  `*Test.java`, `*_test.go`, `*.spec.ts`, `test_*.py`…), em lista configurável.
- **Resolve nomes de variável** — quando o nome vem de `${KAFKA_TOPIC}` ou
  `process.env.QUEUE_URL`, o detector de config cruza os arquivos para achar o valor
  literal. Se não resolver, marca como **não-resolvido** — nunca chuta.
- **Um nó por API** — várias chamadas à mesma API viram **um nó**; as rotas
  (`/charge`, `/refund`) ficam como **detalhe da ligação** (visível ao clicar), sem
  poluir o mapa.

### Técnica e evolução
Começa com **regex** (simples, rápido de pôr de pé) e evolui para **AST/tree-sitter**
(entende estrutura, erra menos) — por ferramenta, no ritmo de cada uma. Os padrões
são **declarativos** (cada padrão é uma entrada de dados com um exemplo que serve de
teste), então o catálogo é auditável e crescer é sempre a mesma operação:

- **nova tecnologia numa linguagem** → nova entrada no catálogo daquela ferramenta;
- **nova linguagem** → nova ferramenta `discovery-<lang>`;
- **mais precisão** → migrar uma ferramenta de regex para AST.

Nada disso quebra o resto do fluxo, porque o **fato** é a interface estável.

---

## Etapa 4 — Analisar

Transforma os **fatos** crus (do discovery) em **entendimento**: para cada
codebase, os componentes, suas integrações (com direção) e o papel do codebase. É a
primeira etapa onde a **IA entra** — até aqui tudo foi determinístico.

**Como faz.** Um **agente analisador por codebase**, em **paralelo** (pool
limitado). Cada agente recebe:

- os **fatos** daquele repo (saída do discovery);
- **ferramentas de leitura** do código (ler arquivo, listar pasta, buscar trecho)
  para investigar quando os fatos não bastam.

E devolve, em formato **estruturado**: os **nós** e **ligações** (cada um com
evidência) + a **descrição do papel** do codebase.

**Regra de ouro.** O agente **nunca afirma uma integração sem evidência**. Marca
contraparte externa como candidata a horizonte. Um fato de **baixa confiança** (do
`discovery-generic`) é dele a responsabilidade de **promover ou descartar** — ele lê
o trecho e decide se é integração real.

### Profundidade da análise (seletor)

O usuário escolhe **quanto esforço** o agente investe, por análise, num seletor no
frontend (**padrão: baixo**). O nível controla **três alavancas juntas** — quanto o
agente lê, qual modelo usa, e quanto ele **raciocina** (reasoning):

| Nível | Leitura de código | Reasoning | Modelo | Custo/tempo |
|---|---|---|---|---|
| **Baixo** (padrão) | pouca ou nenhuma; trabalha **só com os fatos** | mínimo / direto | mais econômico | menor |
| **Médio** | lê **ao redor dos fatos incertos** para confirmar/descartar | moderado, nos pontos de decisão | intermediário | médio |
| **Alto** | **investiga amplamente**; procura integrações que o discovery perdeu; extrai regras de negócio ricas | estendido, antes de cada conclusão | mais forte | maior |

O seletor é o **controle de custo** da análise: em vez de um limite fixo, o usuário
decide o equilíbrio entre rapidez/economia e profundidade. O padrão **baixo** dá um
mapa rápido; sobe-se para médio/alto quando se quer mais.

**O que é reasoning.** É a capacidade do modelo de **pensar passo a passo antes de
responder** — ponderar a evidência, descartar hipóteses, verificar a própria lógica.
Melhora o julgamento em casos ambíguos, ao custo de mais tokens e tempo. Por isso
ele é uma das alavancas que sobem com a profundidade, não um controle à parte.

Nesta etapa, o reasoning ajuda o agente a decidir se um sinal incerto é integração
real, inferir regras de negócio e entender o papel do codebase. (Ele também atua nos
outros pontos de julgamento do fluxo — ambiguidade no Montar e correção no Validar —
ver "Onde a IA decide".)

**Entrada.** Fatos do discovery (por repo) + acesso ao código + nível escolhido.

**Saída.** Por codebase: nós + ligações + papel, tudo com evidência — pronto para a
etapa 5 (montar) juntar.

---

## Etapa 5 — Montar

Junta as análises de todos os codebases (uma por agente) num **grafo único** do
fluxo. É onde as peças soltas viram um mapa conectado.

**Como faz.**
- **Une nós e ligações** de todos os codebases analisados.
- **Aplica IDs canônicos** (`tipo + nome`) — o mesmo recurso citado por dois
  codebases vira **um nó só**. Ex.: o tópico `customer.created` que o
  `customer-service` produz e o `notification-service` consome é o mesmo nó.
- **Casa produtor × consumidor** — pelo nome canônico do tópico/fila, liga quem
  publica a quem consome, com a seta no sentido certo (usa a `direção` do fato).
- **Funde duplicatas exatas** — mesmo ID = mesmo nó; as evidências dos dois lados se
  acumulam.
- **Marca horizontes** — toda contraparte que aparece nas integrações mas **não tem
  codebase no fluxo** vira nó-horizonte (ex.: `payments.acme.com` que alguém chama,
  mas cujo repo não foi fornecido).

**Natureza.** Majoritariamente **determinístico** (junta por nome exato). A IA só
entra **na ambiguidade de identidade**.

**Ambiguidade (o ponto de decisão).** Quando dois recursos são **parecidos mas não
idênticos** (`customer.created` vs `customer-created`, ou nome vindo de variável
não-resolvida), o script não sabe sozinho se são o mesmo. O comportamento **escala
com a profundidade** (seletor da Etapa 4):

- **Baixo** — casa só nomes **idênticos**; parecidos ficam **separados**.
- **Médio / Alto** — a IA (com reasoning) decide os casos parecidos.

**Quando não se resolve** (nível baixo, ou a IA em dúvida): os recursos ficam
**separados, com selo "possível duplicata"** no nó e aviso na área de logs. Seguimos
o princípio **na dúvida, não inferir** — nunca fundir por engano e mentir no mapa. O
usuário pode fundir manualmente se souber que são o mesmo.

**Entrada.** As análises por codebase (nós + ligações + evidência) da Etapa 4.

**Saída.** O grafo único (`Architecture`: nós + ligações + horizontes), pronto para
a Etapa 6 (validar).

---

## Etapa 6 — Validar

Confere o grafo montado contra **regras determinísticas**. É a camada final
anti-alucinação: rejeita o que passou pela IA sem prova.

### Erro duro vs aviso
A validação separa o que **torna o mapa errado** (erro duro — entra no loop de
correção) do que é **legítimo ou apenas incerto** (aviso — só sinaliza, não bloqueia).
Essa separação existe para o loop gastar esforço só no que está de fato errado.

| Regra | Tipo | Por quê |
|---|---|---|
| nó/ligação **sem evidência** | **erro duro** | não dá para provar que existe |
| evidência aponta para `arquivo:linha` **inexistente** | **erro duro** | evidência fabricada |
| **direção incoerente** (ex.: banco "publicando") | **erro duro** | logicamente impossível |
| tópico/fila com **uma ponta só** | aviso | é horizonte — legítimo |
| **possível duplicata** (parecidos não fundidos) | aviso | decidido manter separado e marcar |
| fato de **baixa confiança** / nome não-resolvido | aviso | incerto, mas mostrado |

(Exceção da 1ª regra: infra declarada à mão em `known_infra` não precisa de
evidência.) Regra adicional determinística: garantir **sem duplicata** por ID
canônico.

### Ruído genérico vs recurso não-resolvido
Um sinal pode capturar a **categoria sem o recurso concreto** — "algum S3", "algum
HTTP" — sem nome próprio. O tratamento depende de haver **ação concreta** na
evidência:

- **presença vaga, sem ação** (ex.: só importa a lib de S3, sem nenhuma chamada) →
  é **ruído genérico**, **removido** — não é um componente real, só poluiria o mapa;
- **ação concreta, mas sem nome resolvido** (ex.: `put_object` num bucket cujo nome
  veio de variável não-resolvida) → **mantido** como nó **"recurso não-identificado"**,
  não removido. Há uma integração real ali; perdê-la seria perder sinal em silêncio.

Distinção-chave com o horizonte: **horizonte tem nome** (vindo de evidência) e está
fora do escopo; **não-resolvido tem ação mas não tem nome**; **ruído** não tem nem
nome nem ação. Nenhum horizonte é jamais removido.

> **Limitação conhecida (resolução de nomes).** Hoje a resolução de nomes que vêm
> de variável acontece **dentro de cada repo** (env vars, properties, Terraform do
> próprio repo). Não tratamos ainda **resolução cruzando repositórios** — ex.: o app
> referencia `${QUEUE_URL}` e o valor/declaração está no Terraform de **outro** repo,
> ou em **AWS SSM / Secrets Manager** (valor em runtime, fora do código). Nesses
> casos o recurso fica como **"não-identificado"** ou é declarado à mão em
> `known_infra`. Ligar a referência (chave da variável / SSM) à sua declaração na
> infra fica como **evolução futura**, se necessário.

### O loop validar→corrigir
Se há **erro duro corrigível**, o orquestrador aciona uma correção e **valida de
novo**:

- **a correção age direto no grafo** (com **reasoning**) — ajusta uma direção
  invertida, remove um nó sem evidência — sem re-rodar a análise. Rápido e barato.
- o loop é **limitado** por um máximo de tentativas, garantindo terminação.

**Quando estoura o limite.** O sistema **não trava**: segue para renderizar o grafo
**com os avisos visíveis** na área de logs/erros. Melhor um mapa imperfeito e
sinalizado do que nenhum mapa.

**Avisos** nunca entram no loop — apenas aparecem nos logs e no nó correspondente.

**Entrada.** O grafo único da Etapa 5.

**Saída.** O grafo **validado** (ou validado-com-avisos), pronto para renderizar.

---

## Etapa 7 — Renderizar

Transforma o grafo validado no **artefato final** que o visualizador consome. É o
fim do pipeline do orquestrador.

**O que produz.**
- **O grafo em JSON** (`Architecture`: nós, ligações, evidências, scope, confiança,
  avisos) — a **fonte de verdade**, que o visualizador carrega para desenhar o mapa;
- os **metadados da análise**: nome do fluxo, repos analisados (com SHA), o que ficou
  de fora (área de logs/erros) e o nível de profundidade usado.

**Natureza.** Determinístico — serialização do grafo já pronto. Sem IA.

**Fronteira com o visualizador.** Renderizar produz os **dados**; o visualizador
(frontend) faz a **apresentação** — layout, ícones, arrastar, zoom, clicar→evidência,
busca, exportar PNG/SVG. Essa separação mantém o grafo (dado) independente de como é
mostrado.

- **Layout** (posição dos nós em camadas) é calculado **no frontend** (o visualizador
  usa algoritmos de layout prontos) — o backend não posiciona nós.
- **Exportações estáticas** (PNG/SVG) saem do **visualizador**, sob demanda. Por ora
  renderizar gera **só o JSON**; outros formatos (Mermaid, draw.io) ficam para depois.

**Entrada.** O grafo validado da Etapa 6.

**Saída.** O JSON do grafo + metadados — entregue ao frontend e disponível para
download/reuso.

---

## O visualizador

A tela final é o **mapa do fluxo**: o grafo navegável com os componentes e suas
integrações. Decisões de UX já validadas em protótipo:

### Ícones por nó — híbrido (tecnologia + fallback por tipo)
Cada nó tem um ícone. A escolha espelha a lógica do discovery (específico →
genérico):

- **tecnologia conhecida** → ícone da **marca real** (Kafka, Redis, Postgres,
  Spring…). Reconhecível à primeira vista.
- **tecnologia desconhecida ou horizonte** → ícone do **tipo genérico** (serviço =
  caixa, tópico = broadcast, fila = pilha, banco = cilindro, bucket = balde, cache
  = raio, horizonte = nuvem). O fallback nunca falha.

Fonte dos ícones: **biblioteca de ícones (brands)** para as tecnologias + um **set
próprio** para os tipos genéricos. O mapeamento `tecnologia → ícone` é
**configurável e evoluível**, igual ao catálogo do discovery. Personalização pelo
usuário (trocar o ícone de um nó) fica para fase futura — por ora o ícone é
escolhido automaticamente.

### Cor por categoria
Cor agrupa por categoria (serviço/gateway, tópico/fila, banco/bucket, horizonte),
com legenda fixa. Reforça a leitura junto com o ícone.

### Interação
- **clicar num nó** abre o painel de **evidências** (`arquivo:linha — trecho`) — a
  rastreabilidade é parte da UX; nada está no mapa sem prova;
- **horizonte** aparece com borda tracejada e selo "fronteira — não analisado,
  expansível";
- **busca em linguagem natural** → resposta com evidência + nós destacados no mapa;
- arrastar, zoom, ajustar à tela, **exportar PNG/SVG**.

### Rotas de API
Chamadas para a mesma API são **um nó só**; as rotas (`/charge`, `/refund`) ficam
como detalhe da ligação, visível ao clicar — não poluem o mapa com nós por rota.

---

## O frontend (entrada)

Tela de entrada enxuta, validada em protótipo:

- **input mínimo:** o usuário cola **URLs de repositórios** (uma por linha) ou sobe
  um arquivo `.txt`; um campo **nome do fluxo** é opcional;
- **profundidade da análise:** seletor baixo / médio / alto (padrão **baixo**) —
  controla quanto o agente investiga (ver Etapa 4);
- **um botão** "Iniciar análise" dispara o fluxo inteiro;
- **progresso ao vivo:** status por etapa (coletar → detect stack → discovery →
  analisar → montar → validar → renderizar) com log em tempo real;
- **resultado:** resumo (componentes / integrações / horizontes) + prévia do mapa,
  com acesso ao visualizador completo.

O usuário **não escreve YAML**. A requisição é só `{ flow_name?, repos[] }`; o
`flow.yaml` é uma **representação interna**, montada pelo sistema, editável só por
quem quiser refinar. Dois campos opcionais de refino:

- **`known_infra`** — recursos declarados à mão (uma fila, um banco) que o usuário
  sabe existir mas que talvez não apareçam no código. É a única exceção à regra
  "sem evidência não entra no mapa".
- **`known_external`** — dá um **rótulo legível** a um horizonte (ex.:
  `payments.acme.com` → "Pagamentos (time X)"). Não cria nó nem infere nada; só
  nomeia melhor o que já foi detectado.

---

## A enriquecer (próximos detalhes)

Já cobertos: as 7 etapas, o conceito de horizonte, o discovery por linguagem, o
seletor de profundidade/reasoning, o frontend e o visualizador.

Ainda a detalhar:

- **modelo de dados completo** — o schema de `Node`, `Edge` e `Architecture` (hoje
  temos o `Fato` do discovery; falta formalizar o grafo);
- **definição concreta dos agentes** — prompt, schema de saída e ferramentas de cada;
- **stack e estrutura do projeto** — linguagens, libs, organização de pastas;
- **fases futuras** — memória vetorial / perguntas em linguagem natural (RAG),
  persistência e histórico, resolução de nomes cruzando repositórios (SSM/infra),
  reuso dos agentes fora do Archscan.

---

## Faseamento da implementação

A estratégia é **esqueleto ponta-a-ponta primeiro**: a Fase 1 entrega o fluxo
inteiro funcionando de forma fina (sem IA, sem frontend rico), e cada fase seguinte
**aprofunda** uma camada. Assim há valor visível e validável cedo, com risco baixo.

**Cada fase é um spec SDD.** O desenvolvimento usa Spec-Driven Development: cada
fase começa por uma **especificação** (o quê, por quê, critérios de aceite), depois
um **plano**, depois a **implementação**. O spec de cada fase é derivado das seções
correspondentes deste documento — este documento é a fonte da especificação.

### Fase 0 — Estrutura base do projeto
Criar o esqueleto de diretórios de todo o projeto, antes de qualquer lógica. Define
a **organização** que sustenta as fases seguintes.

**Repositório:** https://github.com/rvechiato/archscan.git

**Organização: monorepo.** Backend e frontend lado a lado, com docs e exemplos no
mesmo repositório — simples de versionar, spec e código juntos, um `docker-compose`
sobe tudo. Stack: **Python + FastAPI** (backend), **React + React Flow** (frontend).

```
archscan/
├── README.md
├── ARCHSCAN.md                 # este documento de design
├── docker-compose.yml          # sobe backend + frontend (e serviços futuros)
├── docs/
│   └── specs/                  # um spec SDD por fase
│
├── backend/
│   ├── pyproject.toml
│   ├── archscan/
│   │   ├── model/              # Fato, Node, Edge, Architecture (o contrato)
│   │   ├── orchestrator/       # o roteiro das 7 etapas (agente orquestrador)
│   │   ├── steps/              # as etapas determinísticas
│   │   │   ├── collect.py          # 1 Coletar
│   │   │   ├── detect_stack.py     # 2 Detect stack
│   │   │   ├── build.py            # 5 Montar
│   │   │   ├── validate.py         # 6 Validar
│   │   │   └── render.py           # 7 Renderizar
│   │   ├── discovery/          # 3 Discovery — uma ferramenta por linguagem
│   │   │   ├── base.py             # contrato do Fato + interface comum
│   │   │   ├── generic.py          # discovery-generic (fallback)
│   │   │   └── (java.py, python.py, node.py — fases futuras)
│   │   ├── agents/             # 4 Analisar — interface Agent agnóstica + adaptadores LLM
│   │   └── api/                # FastAPI: /analyze, /query
│   └── tests/
│
└── frontend/
    ├── package.json
    └── src/
        ├── pages/             # tela de input, tela de resultado
        ├── components/        # visualizador (React Flow), nós, painel de evidências
        ├── icons/             # ícones por tipo/tecnologia
        └── api/               # cliente da API do backend
```

A estrutura espelha o documento: `model/` é o contrato, `steps/` + `discovery/` +
`agents/` são as 7 etapas, `api/` expõe o backend, e o `frontend/` traz input e
visualizador. Pastas de fases futuras (discovery dedicado, etc.) já têm lugar
reservado.

*Entregável:* repositório montado, com a estrutura, configs base (pyproject,
package.json, docker-compose) e um "esqueleto que sobe" — backend e frontend rodam
vazios, sem lógica ainda.

### Fase 1 — Esqueleto ponta-a-ponta (determinístico)
Rodar o fluxo completo via CLI num conjunto de repos e produzir o grafo em JSON,
**sem IA**. É a fundação e o primeiro entregável validável.

- **Modelo de dados** — `Fato`, `Node`, `Edge`, `Architecture` (o contrato estável).
- **Orquestrador mínimo** — o roteiro fixo das 7 etapas, sem pontos de decisão de IA.
- **Etapa 1 Coletar** · **Etapa 2 Detect stack** · **Etapa 3 Discovery** (só
  `discovery-generic`, regex).
- **Etapa 4 Analisar (stand-in determinístico)** — converte os fatos direto em
  nós/ligações (equivalente ao nível "baixo" sem IA), só para fechar a ponta. É
  substituída pelo agente na Fase 2.
- **Etapa 5 Montar** · **Etapa 6 Validar** (regras, sem loop de IA) · **Etapa 7
  Renderizar** (JSON).

*Entregável:* dá os repos → sai o grafo JSON, com horizontes e evidências.
Validável contra um fluxo conhecido.

### Fase 2 — Agentes de IA
Trocar o stand-in pela inteligência real, ancorada em evidência.

- **Agente analisador** (Etapa 4) — prompt + schema + ferramentas de leitura;
  interface `Agent` agnóstica de vendor; nível **baixo**.
- Regra de evidência; promover/descartar fatos de baixa confiança; horizonte.
- **IA nos pontos de julgamento** — ambiguidade no Montar; loop validar→corrigir.

*Entregável:* mapa enriquecido por IA (nível baixo), ainda via CLI.

### Fase 3 — Frontend + Visualizador
Tornar o produto usável por não-desenvolvedores.

- **Frontend** — input (URLs / `.txt`), nome opcional, seletor de profundidade,
  botão, progresso ao vivo (logs), tela de resultado.
- **Visualizador** — grafo com ícones (híbrido), cor por categoria, clicar →
  evidência, horizonte tracejado, zoom/arrastar/export PNG-SVG. (A busca em
  linguagem natural depende da Fase 6 — aqui entra como filtro simples ou desativada.)

*Entregável:* produto ponta-a-ponta com interface.

### Fase 4 — Profundidade média/alta + reasoning
Ativar os níveis mais profundos do seletor.

- Níveis **médio** e **alto** — leitura ampla, reasoning estendido, modelo mais forte.
- Investigação proativa (procurar integrações que o discovery perdeu).

*Entregável:* análise profunda sob demanda.

### Fase 5 — Discovery dedicado por linguagem
Subir a precisão onde está a maioria dos codebases.

- `discovery-java`, `discovery-python`, `discovery-node` (catálogo declarativo).
- Migração de **regex → AST/tree-sitter** por ferramenta.

*Entregável:* detecção de alta precisão nas linguagens principais.

### Fase 6+ — Fases futuras
Os itens já listados em "A enriquecer", cada um seu próprio spec quando chegar a vez:
memória vetorial / perguntas em linguagem natural (RAG), persistência e histórico,
resolução de nomes cruzando repositórios (SSM/infra), reuso dos agentes fora do
Archscan.

### Sequência e dependências
```
Fase 0 (estrutura) → Fase 1 (esqueleto) → Fase 2 (IA) → Fase 3 (frontend/visual)
                                                    → Fase 4 (profundidade)
                                                    → Fase 5 (discovery dedicado) → Fase 6+
```
Fases 4 e 5 são independentes entre si (podem ir em paralelo após a 3). A busca em
linguagem natural do visualizador (Fase 3) só fica completa com a memória vetorial
da Fase 6.

---

*Archscan — Documento de Design. Mapeamento automático de arquitetura a partir de
codebases, com análise por agentes ancorada em evidência.*
