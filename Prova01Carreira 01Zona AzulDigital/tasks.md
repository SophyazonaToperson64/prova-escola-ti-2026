# Tasks — Prova Escola TI 2026

## 1. Objetivo

Implementar o sistema descrito em `spec.md`, respeitando:

* `contract.json`;
* `constitution.md`;
* `spec.md`;
* `plan.md`;
* `tests.md`.

As tarefas devem ser executadas na ordem apresentada, mantendo o comportamento do contrato como referência principal.

---

# 2. Preparação do projeto

## TASK-01 — Analisar contrato e parâmetros

* Ler `contract.json`.
* Ler `params.json`.
* Confirmar os endpoints, campos, códigos HTTP e identificadores de erro.
* Aplicar os valores da variante:

  * tarifa: `600` centavos/hora;
  * fração: `30` minutos;
  * teto: `5000` centavos;
  * tolerância: `0` minutos;
  * porta: `8005`.
* Não criar regras que não estejam previstas nos documentos.

**Critério de conclusão:**

Todos os requisitos necessários para implementação estão identificados antes da criação das funcionalidades.

---

# 3. Estrutura da aplicação

## TASK-02 — Criar estrutura inicial

Criar a estrutura necessária para:

* aplicação HTTP;
* domínio;
* validações;
* serviços;
* persistência;
* tratamento de erros;
* testes.

A organização pode seguir a arquitetura definida no projeto, desde que não altere o contrato externo.

**Critério de conclusão:**

A aplicação inicia corretamente e possui separação suficiente entre entrada HTTP, regras de negócio e armazenamento.

---

# 4. Modelo de bilhete

## TASK-03 — Implementar entidade de bilhete

Representar os dados necessários do bilhete definidos pelo contrato.

O modelo deve suportar os estados:

```text
aberto
fechado
cancelado
```

Deve permitir representar, quando aplicável:

* identificador;
* placa;
* entrada;
* saída;
* status;
* valor da cobrança.

**Critério de conclusão:**

O domínio consegue representar corretamente os três estados e os dados necessários para as operações.

---

# 5. Persistência

## TASK-04 — Implementar armazenamento dos bilhetes

Implementar o mecanismo de persistência necessário para:

* criar bilhetes;
* localizar por ID;
* localizar bilhete aberto por placa;
* listar bilhetes abertos;
* consultar histórico por placa;
* consultar dados necessários ao relatório.

A solução pode utilizar armazenamento em memória ou outra estratégia compatível com o ambiente da prova, desde que preserve o comportamento esperado entre requisições durante a execução da aplicação.

**Critério de conclusão:**

Todas as operações conseguem consultar e alterar os mesmos registros de forma consistente.

---

# 6. Validação

## TASK-05 — Implementar validação de placa

Validar a placa conforme o contrato:

* obrigatória;
* exatamente 7 caracteres;
* somente caracteres alfanuméricos;
* formato em maiúsculas.

Entradas inválidas devem produzir:

```text
422
placa_invalida
```

**Critério de conclusão:**

Os casos válidos e inválidos descritos em `tests.md` são atendidos.

---

## TASK-06 — Implementar validação de entrada

Validar o campo opcional `entrada`.

Quando informado:

* deve possuir formato ISO-8601 válido;
* deve possuir timezone;
* deve ser interpretável como timestamp válido.

Quando não informado:

* utilizar o instante atual conforme contrato.

Entrada inválida deve produzir:

```text
422
entrada_invalida
```

---

## TASK-07 — Garantir ordem das validações

Garantir que:

1. formato da requisição seja validado;
2. somente depois sejam verificadas regras de negócio e conflitos.

Exemplo:

```text
placa inválida + placa já possui bilhete aberto
```

deve resultar em:

```text
422 placa_invalida
```

e não em:

```text
409 bilhete_em_aberto
```

---

# 7. Abertura

## TASK-08 — Implementar criação de bilhete

Implementar a operação de abertura.

Com entrada válida:

* gerar ID único;
* registrar placa;
* registrar entrada;
* iniciar status como `aberto`;
* retornar o resultado conforme contrato.

Resposta de sucesso:

```text
201
```

---

## TASK-09 — Implementar regra de um bilhete aberto por placa

Antes de criar um novo bilhete:

* verificar se já existe bilhete `aberto` para a placa;
* se existir, rejeitar a operação.

Erro:

```text
409
bilhete_em_aberto
```

Após encerramento ou cancelamento, a placa pode possuir novo bilhete aberto.

---

# 8. Encerramento

## TASK-10 — Implementar localização por ID

Permitir localizar bilhetes pelo identificador.

Quando o ID não existir:

```text
404
bilhete_nao_encontrado
```

---

## TASK-11 — Implementar encerramento

Permitir encerramento de bilhete aberto.

Ao encerrar:

* registrar saída;
* calcular duração;
* calcular cobrança;
* alterar status para `fechado`.

Não permitir encerramento de bilhete já fechado.

Nesse caso:

```text
409
bilhete_ja_encerrado
```

Qualquer outro conflito de estado deve utilizar somente o comportamento definido pelo contrato.

---

# 9. Cobrança

## TASK-12 — Implementar cálculo de duração

Calcular:

```text
duração = saída - entrada
```

A duração deve ser calculada utilizando timestamps reais.

---

## TASK-13 — Implementar tolerância

Aplicar:

```text
TOLERANCIA_MINUTOS = 0
```

Com essa variante:

* duração `0` → sem cobrança;
* duração maior que `0` → cobrança normal.

Não subtrair a tolerância da duração antes do cálculo das frações.

---

## TASK-14 — Implementar cálculo de frações

Utilizar:

```text
frações = ceil(duração_em_minutos / 30)
```

Exemplos obrigatórios:

| Duração | Frações |
| ------: | ------: |
|   0 min |       0 |
|   1 min |       1 |
|  29 min |       1 |
|  30 min |       1 |
|  31 min |       2 |
|  59 min |       2 |
|  60 min |       2 |
|  61 min |       3 |
|  90 min |       3 |
|  91 min |       4 |

---

## TASK-15 — Implementar valor por fração

A tarifa da variante é:

```text
600 centavos/hora
```

Como a fração é de 30 minutos:

```text
valor da fração = 300 centavos
```

Os cálculos devem utilizar inteiros.

---

## TASK-16 — Implementar teto da cobrança

Aplicar:

```text
valor_final = min(valor_bruto, 5000)
```

Verificar obrigatoriamente:

```text
16 frações → 4800
17 frações → 5000
```

Valores acima do teto continuam limitados a `5000` centavos.

---

# 10. Listagem

## TASK-17 — Implementar listagem de bilhetes abertos

Retornar somente bilhetes com status:

```text
aberto
```

Ordenar do mais novo para o mais antigo.

A ordenação deve utilizar informação temporal, e não depender da numeração do ID.

Quando não houver bilhetes abertos:

```json
[]
```

---

# 11. Cancelamento

## TASK-18 — Implementar cancelamento

Permitir cancelamento somente de bilhete aberto.

Ao cancelar:

```text
status = cancelado
```

Bilhete cancelado não deve gerar cobrança.

---

## TASK-19 — Implementar erros de cancelamento

ID inexistente:

```text
404
bilhete_nao_encontrado
```

Bilhete que não está aberto:

```text
409
bilhete_nao_aberto
```

Não criar identificadores de erro adicionais sem previsão contratual.

---

# 12. Histórico

## TASK-20 — Implementar histórico por placa

Consultar os bilhetes associados a uma placa.

O histórico deve incluir os estados previstos:

* aberto;
* fechado;
* cancelado.

Ordenar do mais novo para o mais antigo.

Quando não houver histórico:

```json
[]
```

---

# 13. Relatório

## TASK-21 — Implementar relatório diário

Implementar o relatório conforme o contrato.

A resposta deve conter os campos definidos:

* `data`;
* `total_bilhetes`;
* `faturamento_centavos`;
* `tempo_medio_minutos`.

As regras de inclusão de cada indicador devem seguir exatamente o contrato.

Não criar uma interpretação própria para campos cuja semântica já esteja definida no contrato.

---

## TASK-22 — Implementar cálculo da média

Para `tempo_medio_minutos`:

* considerar somente os bilhetes encerrados elegíveis para o dia solicitado;
* calcular a média conforme contrato;
* aplicar arredondamento half-up.

Exemplos:

```text
10,4 → 10
10,5 → 11
10,6 → 11
```

---

## TASK-23 — Respeitar separação por data

Garantir que um bilhete encerrado em outra data não seja indevidamente incluído no cálculo referente ao dia consultado.

---

# 14. Tratamento de erros

## TASK-24 — Padronizar respostas de erro

Garantir que os erros previstos retornem:

* código HTTP correto;
* identificador de erro correto;
* estrutura compatível com o contrato.

Não alterar nomes de erros para facilitar a implementação.

---

# 15. Testes

## TASK-25 — Implementar testes de abertura

Cobrir:

* placa válida;
* placa inválida;
* tamanho menor;
* tamanho maior;
* caracteres inválidos;
* entrada válida;
* entrada inválida;
* entrada sem timezone;
* geração de ID.

---

## TASK-26 — Implementar testes de estado

Cobrir:

* aberto → fechado;
* aberto → cancelado;
* tentativa de fechar inexistente;
* tentativa de fechar já fechado;
* tentativa de cancelar inexistente;
* tentativa de cancelar não aberto;
* novo bilhete após fechamento;
* novo bilhete após cancelamento.

---

## TASK-27 — Implementar testes de cobrança

Cobrir todas as fronteiras:

```text
0
1
29
30
31
59
60
61
90
91
```

Também cobrir:

```text
16 frações = 4800
17 frações = 5000
```

e valores superiores ao teto.

---

## TASK-28 — Implementar testes de listagem e histórico

Validar:

* filtro por estado;
* histórico completo;
* lista vazia;
* ordenação temporal;
* independência dos IDs.

---

## TASK-29 — Implementar testes do relatório

Validar:

* estrutura da resposta;
* separação por data;
* faturamento;
* média;
* somente bilhetes elegíveis para a média;
* arredondamento half-up;
* ausência de dados elegíveis.

---

## TASK-30 — Implementar testes de contrato HTTP

Validar:

* métodos;
* endpoints;
* códigos HTTP;
* payloads;
* campos;
* erros;
* status de sucesso;
* status de erro.

---

# 16. Infraestrutura

## TASK-31 — Configurar porta

Configurar a aplicação para utilizar:

```text
8005
```

A porta não deve ficar fixada em `8080` quando o ambiente da prova exigir `8005`.

---

## TASK-32 — Configurar manifesto

Garantir que todas as dependências necessárias estejam declaradas no arquivo de dependências correspondente ao projeto.

O projeto deve poder ser instalado e executado sem dependências ocultas na máquina do desenvolvedor.

---

## TASK-33 — Configurar containerização

Se o projeto utilizar Docker:

* garantir que a imagem seja construída;
* garantir que a aplicação inicie;
* expor a porta necessária;
* garantir que a aplicação responda às requisições.

---

## TASK-34 — Documentar execução

O `README` deve explicar de forma objetiva:

* como instalar dependências;
* como executar a aplicação;
* como executar os testes;
* qual porta utilizar;
* como realizar uma execução básica do sistema.

---

# 17. Revisão final

## TASK-35 — Revisar contra o contrato

Comparar a implementação com `contract.json`.

Verificar:

* endpoints;
* métodos;
* campos;
* tipos;
* códigos HTTP;
* erros;
* estados;
* regras de negócio.

---

## TASK-36 — Revisar casos de borda

Executar/revisar principalmente:

* 0 minutos;
* 1 minuto;
* 29 minutos;
* 30 minutos;
* 31 minutos;
* 59 minutos;
* 60 minutos;
* 61 minutos;
* 90 minutos;
* 91 minutos;
* 16 frações;
* 17 frações;
* placa duplicada;
* placa inválida + conflito;
* bilhete inexistente;
* bilhete já encerrado;
* bilhete cancelado;
* lista vazia;
* histórico vazio;
* média com `.4`;
* média com `.5`;
* média com `.6`.

---

## TASK-37 — Revisar valores monetários

Garantir que:

* todos os valores sejam tratados em centavos;
* não exista cálculo monetário baseado em `float`;
* o teto seja aplicado;
* o valor da fração seja `300` centavos.

---

## TASK-38 — Revisar ordenação

Garantir que:

* listagem de abertos seja do mais novo para o mais antigo;
* histórico seja do mais novo para o mais antigo;
* ordenação não dependa exclusivamente do ID.

---

## TASK-39 — Revisar estados

Confirmar que:

```text
aberto → fechado
aberto → cancelado
```

são as transições válidas previstas.

Confirmar também que:

* fechado não volta para aberto;
* cancelado não volta para aberto;
* bilhete fechado não pode ser encerrado novamente;
* bilhete não aberto não pode ser cancelado.

---

## TASK-40 — Revisão final da entrega

Antes de finalizar:

1. confirmar presença dos cinco documentos;
2. conferir consistência entre `constitution.md`, `spec.md`, `plan.md`, `tests.md` e `tasks.md`;
3. verificar se nenhuma regra foi inventada;
4. verificar se nenhuma regra do contrato foi omitida;
5. verificar se a porta é `8005`;
6. verificar se os valores da variante estão corretos;
7. executar os testes automatizados;
8. verificar o README;
9. verificar o manifesto de dependências;
10. verificar a possibilidade de execução limpa do projeto.

---

# 18. Ordem resumida de execução

A sequência recomendada é:

```text
TASK-01
   ↓
TASK-02
   ↓
TASK-03 / TASK-04
   ↓
TASK-05 / TASK-06 / TASK-07
   ↓
TASK-08 / TASK-09
   ↓
TASK-10 / TASK-11
   ↓
TASK-12 → TASK-16
   ↓
TASK-17
   ↓
TASK-18 / TASK-19
   ↓
TASK-20
   ↓
TASK-21 → TASK-23
   ↓
TASK-24
   ↓
TASK-25 → TASK-30
   ↓
TASK-31 → TASK-34
   ↓
TASK-35 → TASK-40
```

A ordem não obriga uma tecnologia específica. O objetivo é reduzir ambiguidades e garantir que cada requisito seja implementado e validado antes da revisão final.
