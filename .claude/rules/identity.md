# Regra — Identidade canônica de nós

Aplicável à Etapa 5 (Montar). Define como IDs são gerados e como duplicatas são tratadas.

## Formato do ID

O ID canônico de um nó é sempre `"tipo:nome"` — sem espaços, sem acentos no tipo.

```
"topic:customer.created"        # tópico Kafka
"service:customer-service"      # serviço mapeado de um repo
"api:api.payments.acme.com"     # endpoint externo chamado
"horizon:api.payments.acme.com" # mesmo host, mas sem repo analisado
```

## Fusão de nós

- Mesmo ID → mesmo nó. As `evidencias` dos dois lados se acumulam na lista.
- IDs diferentes → nós separados, mesmo que os nomes sejam parecidos.

## Tratamento de nomes parecidos (Fase 1)

Na Fase 1, a fusão é **só por nome idêntico**. Nomes parecidos mas não idênticos
(`customer.created` vs `customer-created`) ficam como nós separados, cada um
recebendo o aviso `"possível duplicata de <outro_id>"` no campo `avisos`.

Na Fase 2, a IA (com reasoning) decide os casos parecidos antes de montar.

## Quando marcar como horizonte

Um nó recebe `escopo = "horizonte"` quando a contraparte da integração não tem
nenhum repo analisado no fluxo. Na Fase 1, a heurística é:

- O nome do recurso contém `.` (sugere host externo: `api.acme.com`)
- O nome não coincide com nenhum `repo` do manifesto

Nós-horizonte **nunca são removidos** — nem pela validação, nem pela limpeza de
ruído. São expansíveis: se o repo for adicionado depois, o nó é promovido a interno.
