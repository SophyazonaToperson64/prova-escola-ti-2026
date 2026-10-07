# Tests — Prova Escola TI 2026

## 1. Objetivo

Este documento define os cenários de teste que devem validar o comportamento especificado para o sistema de gestão de bilhetes.

Os testes devem verificar principalmente:

* contrato HTTP;
* validações de entrada;
* regras de estado;
* unicidade de bilhete aberto por placa;
* cálculo de cobrança;
* tolerância;
* teto diário/por bilhete conforme contrato;
* listagem e ordenação;
* histórico;
* relatório diário;
* tratamento dos erros definidos pelo contrato.

Os testes devem utilizar os valores da variante atual definidos em `params.json`.

---

## 2. Valores da variante

| Parâmetro        |         Valor |
| ---------------- | ------------: |
| Tarifa por hora  |  600 centavos |
| Fração           |    30 minutos |
| Valor por fração |  300 centavos |
| Tolerância       |     0 minutos |
| Teto             | 5000 centavos |
| Porta do serviço |          8005 |
| Fuso horário     |      `-03:00` |

O valor por fração deve ser tratado como inteiro:

```text
600 / 2 = 300 centavos
```

Não devem existir cálculos monetários utilizando ponto flutuante.

---

# 3. Testes de abertura de bilhete

## T01 — Abertura válida somente com placa

**Entrada:**

```json
{
  "placa": "ABC1234"
}
```

**Resultado esperado:**

* HTTP `201`;
* bilhete criado;
* resposta contém `id`;
* resposta contém `placa`;
* `status` é `aberto`;
* `entrada` é preenchida pelo sistema quando não fornecida.

---

## T02 — Abertura válida com entrada informada

**Entrada:**

```json
{
  "placa": "ABC1234",
  "entrada": "2026-10-07T10:00:00-03:00"
}
```

**Resultado esperado:**

* HTTP `201`;
* timestamp informado é preservado;
* status inicial é `aberto`.

---

## T03 — Placa com exatamente 7 caracteres

Exemplo:

```text
ABC1234
```

**Resultado esperado:**

* abertura aceita;
* HTTP `201`.

---

## T04 — Placa com menos de 7 caracteres

Exemplo:

```text
ABC123
```

**Resultado esperado:**

* HTTP `422`;
* erro `placa_invalida`;
* nenhum bilhete deve ser criado.

---

## T05 — Placa com mais de 7 caracteres

Exemplo:

```text
ABC12345
```

**Resultado esperado:**

* HTTP `422`;
* erro `placa_invalida`.

---

## T06 — Placa contendo caracteres inválidos

Exemplos:

```text
ABC-123
ABC 123
ABC@123
```

**Resultado esperado:**

* HTTP `422`;
* erro `placa_invalida`.

---

## T07 — Placa em minúsculas

Exemplo:

```text
abc1234
```

**Resultado esperado:**

* HTTP `422`;
* erro `placa_invalida`.

A validação não deve transformar silenciosamente a entrada para maiúsculas.

---

## T08 — Entrada com formato ISO-8601 válido

Exemplo:

```text
2026-10-07T10:00:00-03:00
```

**Resultado esperado:**

* HTTP `201`;
* timestamp aceito.

---

## T09 — Entrada sem timezone

Exemplo:

```text
2026-10-07T10:00:00
```

**Resultado esperado:**

* HTTP `422`;
* erro `entrada_invalida`.

---

## T10 — Entrada com timestamp inválido

Exemplo:

```text
2026-99-99T99:99:99-03:00
```

**Resultado esperado:**

* HTTP `422`;
* erro `entrada_invalida`.

---

# 4. Ordem de validação

## T11 — Requisição inválida deve ser rejeitada antes do conflito

Primeiro criar um bilhete aberto para:

```text
ABC1234
```

Depois enviar nova abertura para a mesma placa, mas com placa inválida.

**Resultado esperado:**

* HTTP `422`;
* erro de validação correspondente;
* não deve retornar `409 bilhete_em_aberto`.

A validação da requisição possui prioridade sobre regras de conflito de estado.

---

# 5. Regra de um único bilhete aberto por placa

## T12 — Não permitir dois bilhetes abertos para a mesma placa

1. Criar bilhete para `ABC1234`.
2. Tentar criar outro bilhete para `ABC1234`.

**Resultado esperado da segunda operação:**

* HTTP `409`;
* erro `bilhete_em_aberto`.

---

## T13 — Permitir novo bilhete depois do encerramento

1. Criar bilhete para `ABC1234`.
2. Encerrar o bilhete.
3. Criar novo bilhete para `ABC1234`.

**Resultado esperado:**

* terceira operação retorna HTTP `201`;
* novo bilhete inicia como `aberto`.

---

## T14 — Permitir novo bilhete depois do cancelamento

1. Criar bilhete para `ABC1234`.
2. Cancelar o bilhete.
3. Criar novo bilhete para `ABC1234`.

**Resultado esperado:**

* terceira operação retorna HTTP `201`.

---

# 6. Encerramento

## T15 — Encerrar bilhete aberto

Criar um bilhete aberto e executar a operação de encerramento.

**Resultado esperado:**

* HTTP de sucesso definido pelo contrato;
* status alterado para `fechado`;
* saída registrada;
* valor cobrado calculado conforme a regra de cobrança.

---

## T16 — Encerrar bilhete inexistente

Utilizar um ID inexistente.

**Resultado esperado:**

* HTTP `404`;
* erro `bilhete_nao_encontrado`.

---

## T17 — Encerrar bilhete já encerrado

1. Criar bilhete.
2. Encerrar.
3. Tentar encerrar novamente.

**Resultado esperado:**

* HTTP `409`;
* erro `bilhete_ja_encerrado`.

---

# 7. Cancelamento

## T18 — Cancelar bilhete aberto

Criar um bilhete aberto e cancelar.

**Resultado esperado:**

* operação aceita;
* status final `cancelado`;
* bilhete não deve gerar cobrança.

---

## T19 — Cancelar bilhete inexistente

**Resultado esperado:**

* HTTP `404`;
* erro `bilhete_nao_encontrado`.

---

## T20 — Cancelar bilhete que não está aberto

Tentar cancelar um bilhete já encerrado ou em estado diferente de `aberto`.

**Resultado esperado:**

* HTTP `409`;
* erro `bilhete_nao_aberto`.

---

# 8. Cálculo da cobrança

A cobrança deve utilizar:

```text
duração = saída - entrada
```

Com tolerância igual a `0`, qualquer duração positiva deve ser cobrada.

A quantidade de frações é:

```text
frações = ceil(duração_em_minutos / 30)
```

Cada fração vale:

```text
300 centavos
```

O valor final é limitado a:

```text
5000 centavos
```

---

## T21 — Duração zero

Entrada e saída no mesmo instante.

**Resultado esperado:**

```text
0 centavos
```

---

## T22 — 1 minuto

**Resultado esperado:**

```text
1 fração
300 centavos
```

---

## T23 — 29 minutos

**Resultado esperado:**

```text
1 fração
300 centavos
```

---

## T24 — Exatamente 30 minutos

**Resultado esperado:**

```text
1 fração
300 centavos
```

---

## T25 — 31 minutos

**Resultado esperado:**

```text
2 frações
600 centavos
```

---

## T26 — 59 minutos

**Resultado esperado:**

```text
2 frações
600 centavos
```

---

## T27 — Exatamente 60 minutos

**Resultado esperado:**

```text
2 frações
600 centavos
```

---

## T28 — 61 minutos

**Resultado esperado:**

```text
3 frações
900 centavos
```

---

## T29 — 90 minutos

**Resultado esperado:**

```text
3 frações
900 centavos
```

---

## T30 — 91 minutos

**Resultado esperado:**

```text
4 frações
1200 centavos
```

---

# 9. Tolerância

A variante possui:

```text
TOLERANCIA_MINUTOS = 0
```

Portanto:

## T31 — Duração igual à tolerância

Duração:

```text
0 minutos
```

**Resultado esperado:**

```text
0 centavos
```

---

## T32 — Duração imediatamente superior à tolerância

Duração:

```text
1 minuto
```

**Resultado esperado:**

```text
300 centavos
```

Não deve ser aplicada nenhuma tolerância adicional.

A tolerância também não deve ser subtraída da duração antes do cálculo das frações.

---

# 10. Teto de cobrança

O teto é:

```text
5000 centavos
```

Como cada fração custa `300` centavos:

```text
16 frações = 4800 centavos
17 frações = 5100 centavos
```

## T33 — Valor abaixo do teto

Duração correspondente a 16 frações.

**Resultado esperado:**

```text
4800 centavos
```

---

## T34 — Valor ultrapassa o teto

Duração correspondente a 17 frações.

Valor bruto:

```text
5100 centavos
```

**Resultado esperado:**

```text
5000 centavos
```

---

## T35 — Duração muito superior ao teto

Utilizar uma duração que gere muitas frações.

**Resultado esperado:**

* valor final continua sendo `5000` centavos;
* nunca deve ultrapassar o teto.

---

# 11. Valores monetários

## T36 — Valor deve ser representado em centavos

A API e o domínio devem trabalhar com valores como:

```json
{
  "valor_centavos": 300
}
```

Não utilizar:

```json
{
  "valor": 3.0
}
```

quando o contrato exigir o campo em centavos.

---

## T37 — Cálculo sem erro de ponto flutuante

Executar cobranças em diferentes durações e verificar que os resultados permanecem inteiros e exatos.

Exemplos:

```text
1 fração  -> 300
2 frações -> 600
3 frações -> 900
16 frações -> 4800
17 frações -> 5000
```

---

# 12. Listagem de bilhetes abertos

## T38 — Listar bilhetes abertos

Criar múltiplos bilhetes em estado `aberto`.

**Resultado esperado:**

* somente bilhetes abertos são retornados;
* bilhetes fechados não aparecem;
* bilhetes cancelados não aparecem.

---

## T39 — Lista vazia

Sem nenhum bilhete aberto.

**Resultado esperado:**

```json
[]
```

---

## T40 — Ordenação dos bilhetes abertos

Criar bilhetes em momentos diferentes.

**Resultado esperado:**

* bilhetes mais novos aparecem primeiro;
* a ordenação deve utilizar informação temporal do bilhete;
* não assumir que o maior ID representa necessariamente o bilhete mais novo.

---

# 13. Histórico por placa

## T41 — Histórico contém todos os estados

Para uma mesma placa:

1. criar bilhete;
2. encerrar um bilhete;
3. criar outro;
4. cancelar outro.

**Resultado esperado:**

O histórico deve permitir visualizar os bilhetes da placa independentemente do estado.

---

## T42 — Histórico de placa sem registros

Consultar uma placa sem histórico.

**Resultado esperado:**

```json
[]
```

---

## T43 — Histórico ordenado do mais novo para o mais antigo

Criar vários bilhetes para a mesma placa em momentos diferentes.

**Resultado esperado:**

* maior timestamp de entrada primeiro;
* menor timestamp de entrada depois.

A ordenação não deve depender exclusivamente do ID.

---

# 14. Relatório diário

## T44 — Relatório para uma data válida

Consultar o relatório de uma data válida conforme o contrato.

**Resultado esperado:**

A resposta contém os campos definidos pelo contrato:

* `data`;
* `total_bilhetes`;
* `faturamento_centavos`;
* `tempo_medio_minutos`.

---

## T45 — Faturamento considera os bilhetes conforme a regra do relatório

Criar bilhetes em diferentes estados e verificar que o faturamento segue exatamente a definição estabelecida pelo contrato.

Não incluir regras adicionais que não estejam definidas no contrato.

---

## T46 — Média considera somente bilhetes encerrados no dia

Criar:

* bilhete encerrado;
* bilhete ainda aberto;
* bilhete cancelado.

**Resultado esperado:**

O cálculo de `tempo_medio_minutos` considera somente os bilhetes encerrados que pertencem ao dia solicitado, conforme contrato.

---

## T47 — Média sem bilhetes encerrados

Quando não houver bilhetes encerrados elegíveis para o cálculo da média:

* seguir exatamente a representação definida pelo contrato;
* não inventar valor ou comportamento adicional.

---

# 15. Arredondamento da média

O cálculo da média deve utilizar arredondamento:

```text
0,5 para cima
```

## T48 — Média com parte decimal menor que 0,5

Exemplo conceitual:

```text
10,4
```

**Resultado esperado:**

```text
10
```

---

## T49 — Média exatamente com 0,5

Exemplo:

```text
10,5
```

**Resultado esperado:**

```text
11
```

---

## T50 — Média com parte decimal maior que 0,5

Exemplo:

```text
10,6
```

**Resultado esperado:**

```text
11
```

O arredondamento deve ser half-up, e não o comportamento de arredondamento bancário.

---

# 16. Isolamento por data

## T51 — Bilhete encerrado em outro dia não entra na média do dia consultado

Criar bilhetes encerrados em datas diferentes.

Consultar apenas uma das datas.

**Resultado esperado:**

* somente os bilhetes pertencentes ao dia solicitado são considerados no cálculo correspondente.

---

# 17. Contrato de erros HTTP

## T52 — Erro de validação

Quando a entrada violar uma regra de formato:

**Resultado esperado:**

```text
HTTP 422
```

e o identificador de erro correspondente definido pelo contrato.

---

## T53 — Recurso inexistente

Quando o ID do bilhete não existir:

```text
HTTP 404
bilhete_nao_encontrado
```

---

## T54 — Conflito de estado

Quando a operação for válida em formato, mas proibida pelo estado atual:

```text
HTTP 409
```

utilizando somente o erro definido pelo contrato para aquele caso.

---

# 18. Prioridade dos erros

## T55 — Formato inválido possui prioridade sobre estado

Enviar uma requisição que:

* possui dados inválidos;
* também poderia gerar conflito com um bilhete existente.

**Resultado esperado:**

* retornar erro de validação `422`;
* não retornar conflito `409`.

---

# 19. Identificadores

## T56 — ID é retornado após criação

Toda criação bem-sucedida deve retornar um identificador do bilhete.

---

## T57 — IDs são únicos

Criar vários bilhetes.

**Resultado esperado:**

* nenhum bilhete deve receber o mesmo ID de outro bilhete existente.

A estratégia interna de geração do ID não é imposta pelos testes, desde que respeite o contrato.

---

# 20. Data e hora

## T58 — Preservação do timezone

Utilizar entrada:

```text
2026-10-07T10:00:00-03:00
```

**Resultado esperado:**

O timestamp deve ser interpretado corretamente respeitando o offset informado.

---

## T59 — Duração calculada corretamente entre timestamps

Utilizar timestamps com horários diferentes e verificar que a duração utilizada na cobrança corresponde à diferença real entre entrada e saída.

Não calcular duração apenas comparando strings.

---

# 21. Isolamento dos testes

## T60 — Estado de um teste não deve contaminar outro

Cada cenário deve possuir estado inicial controlado.

A execução de um teste não deve depender da ordem em que outro teste foi executado.

---

# 22. Infraestrutura e execução

## T61 — Serviço inicia na porta definida

O serviço deve estar disponível na porta:

```text
8005
```

Não utilizar `8080` como porta final quando o contrato da variante exigir `8005`.

---

## T62 — Dependências declaradas

Todas as dependências necessárias para executar a aplicação devem estar declaradas no manifesto correspondente ao projeto.

---

## T63 — Containerização

Caso a entrega utilize containerização, a aplicação deve iniciar corretamente dentro do container e disponibilizar o serviço na porta definida.

---

## T64 — Testes automatizados executáveis

Os testes automatizados devem poder ser executados pelo mecanismo padrão do projeto sem depender de alterações manuais no código.

---

# 23. Matriz de rastreabilidade

| Requisito                   | Testes principais |
| --------------------------- | ----------------- |
| Criar bilhete               | T01–T10           |
| Validação de placa          | T03–T07           |
| Validação de entrada        | T08–T10           |
| Validação antes do conflito | T11, T55          |
| Um aberto por placa         | T12–T14           |
| Encerramento                | T15–T17           |
| Cancelamento                | T18–T20           |
| Cobrança                    | T21–T30           |
| Tolerância                  | T31–T32           |
| Teto                        | T33–T35           |
| Valores em centavos         | T36–T37           |
| Bilhetes abertos            | T38–T40           |
| Histórico                   | T41–T43           |
| Relatório                   | T44–T47           |
| Média                       | T48–T50           |
| Filtro por data             | T51               |
| Erros HTTP                  | T52–T55           |
| Identificadores             | T56–T57           |
| Data/hora                   | T58–T59           |
| Isolamento                  | T60               |
| Porta                       | T61               |
| Manifesto                   | T62               |
| Container                   | T63               |
| Execução dos testes         | T64               |

---

# 24. Regras para implementação dos testes

Os testes devem:

1. respeitar exatamente os códigos HTTP definidos pelo contrato;
2. respeitar exatamente os identificadores de erro definidos pelo contrato;
3. validar valores monetários em centavos;
4. validar os limites de cobrança;
5. validar as transições de estado;
6. validar a ordenação temporal;
7. validar a precedência de erros de validação;
8. evitar dependência entre testes;
9. evitar assumir uma estratégia específica de geração de IDs;
10. não criar regras de negócio que não estejam presentes no contrato ou na especificação.

Os casos de borda devem ter prioridade, pois são os cenários mais suscetíveis a divergências entre uma implementação aparentemente correta e o comportamento esperado pelos testes ocultos.
