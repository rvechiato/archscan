# Regra — Validação do grafo

Aplicável à Etapa 6 (Validar). Define o que é erro duro, o que é aviso e como
cada caso é tratado.

## Erros duros (bloqueantes)

Entram no loop de correção (Fase 2) ou são removidos diretamente (Fase 1).

| Regra | Ação |
|---|---|
| Nó ou ligação sem `evidencias` | remove do grafo + registra em `Architecture.erros` |
| `evidencia` aponta para `arquivo:linha` inexistente no disco | remove + registra |
| Direção semanticamente impossível (ex.: `database` com direção `produz`) | remove + registra |

Exceção: nós declarados em `known_infra` não precisam de evidência de código.

## Avisos (não-bloqueantes)

Nunca entram no loop — apenas aparecem em `Architecture.avisos` e no nó correspondente.

| Regra | Aviso gerado |
|---|---|
| Tópico ou fila com produtor mas sem consumidor (ou vice-versa) | `"ponta única — pode ser horizonte"` |
| Nós com ID diferente mas nome parecido não fundidos | `"possível duplicata de <outro_id>"` |
| Fato de `confianca: "baixa"` mantido no grafo | `"confiança baixa — lib não identificada"` |
| Nome do recurso não-resolvido (veio de variável não expandida) | `"nome não-resolvido"` |

## Distinção ruído × não-resolvido × horizonte

- **Ruído genérico** — presença da lib sem nenhuma ação concreta (ex.: só importa boto3, sem nenhuma chamada). É **removido** silenciosamente; não gera aviso.
- **Recurso não-identificado** — há ação concreta (`put_object`, `publish`) mas o nome veio de variável não-resolvida. É **mantido** com aviso `"nome não-resolvido"`. Perder essa integração seria perder sinal real.
- **Horizonte** — tem nome (vindo de evidência), mas o repo não foi fornecido. **Nunca removido**.

## Comportamento na Fase 1

Sem loop de correção: erros duros são removidos imediatamente. O grafo resultante
pode ser parcial — é entregue com `Architecture.erros` preenchido, não abortado.

O loop de correção por IA (máximo de tentativas configurável) entra na Fase 2.
