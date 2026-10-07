# Constitution — Sistema de Gestão de Bilhetes

## 1. Objetivo

Este documento define as regras globais e invariantes que devem ser respeitadas por toda a implementação do sistema.

A implementação deve seguir o contrato funcional fornecido pela prova e não deve criar comportamentos, campos, endpoints, códigos de erro ou regras de negócio que não estejam definidos no contrato ou explicitamente especificados nos documentos desta prova.

Em caso de conflito entre decisões de implementação e o contrato funcional, o contrato deve prevalecer.

---

## 2. Contrato da API

* Os endpoints devem utilizar exatamente os métodos HTTP definidos no contrato.
* Os caminhos dos endpoints devem ser mantidos exatamente como especificados.
* Os nomes dos campos de request e response devem ser preservados.
* Os códigos HTTP devem seguir o contrato.
* Os códigos de erro e seus identificadores devem seguir exatamente os valores definidos no contrato.
* Não devem ser criados códigos de erro alternativos para situações que já possuem comportamento definido pelo contrato.
* Não devem ser adicionados campos obrigatórios que não estejam previstos no contrato.
* Exemplos inconsistentes não devem substituir regras explicitamente definidas no contrato.

---

## 3. Variante da Prova

A implementação deve utilizar os valores da variante fornecida:

| Parâmetro              |  Valor |
| ---------------------- | -----: |
| `TARIFA_HORA_CENTAVOS` |  `600` |
| `FRACAO_MINUTOS`       |   `30` |
| `TETO_DIARIO_CENTAVOS` | `5000` |
| `TOLERANCIA_MINUTOS`   |    `0` |
| `PORTA_SERVICO`        | `8005` |

Esses valores não devem ser substituídos por valores de exemplos de outras variantes.

A aplicação deve permanecer configurável de acordo com os parâmetros da variante quando isso for necessário para a execução da infraestrutura da prova.

---

## 4. Valores Monetários
Todos os valores monetários devem ser representados como inteiros em centavos.

Nesta variante:

* `600` centavos representam R$ 6,00 por hora;
* `300` centavos representam R$ 3,00 por fração de 30 minutos;
* `5000` centavos representam o teto de R$ 50,00 por bilhete.

Não utilizar valores monetários em ponto flutuante.

O cálculo da cobrança deve produzir resultados determinísticos em centavos.

---

## 5. Tempo e Datas

* Datas e horários devem utilizar ISO-8601.
* O fuso horário utilizado pelo sistema é `-03:00`.
* Quando o campo `entrada` for omitido na abertura de um bilhete, deve ser utilizado o instante atual.
* Quando `entrada` for fornecida, o instante informado deve ser utilizado.
* O valor de `entrada` fornecido pelo cliente deve permitir a execução determinística dos cenários de teste.
* Cálculos de duração devem ser realizados a partir dos instantes de entrada e saída.
* Não devem ser introduzidas regras de horário ou arredondamento temporal que não estejam previstas no contrato.

---

## 6. Ordem de Validação

A validação estrutural e de formato deve ocorrer antes da avaliação das regras de negócio.

A ordem geral deve ser:

1. validar presença e formato dos dados;
2. validar os valores permitidos pelo contrato;
3. somente depois avaliar conflitos, existência e estado do recurso.

Consequentemente, quando uma requisição simultaneamente possuir dados inválidos e uma possível condição de conflito, o erro de validação deve prevalecer.

Exemplo:

* uma placa já possui bilhete aberto;
* uma nova requisição utiliza uma placa inválida.

O resultado deve ser `422` referente à placa inválida, e não `409` referente ao bilhete aberto.

---

## 7. Estados do Bilhete

Os estados válidos de um bilhete são exclusivamente:

* `aberto`;
* `encerrado`;
* `cancelado`.

A abertura de um novo bilhete inicia seu estado como `aberto`.

As transições válidas são:

```text
                 ┌───────────┐
                 │   aberto  │
                 └─────┬─────┘
                       │
              ┌────────┴────────┐
              ↓                 ↓
        ┌───────────┐     ┌───────────┐
        │ encerrado │     │ cancelado │
        └───────────┘     └───────────┘
```

Somente um bilhete em estado `aberto` pode ser encerrado ou cancelado.

Depois de encerrado ou cancelado, o bilhete não deve retornar ao estado `aberto`.

Os comportamentos e códigos de erro específicos de cada operação devem seguir o contrato.

---

## 8. Unicidade de Bilhete Aberto por Placa

Uma mesma placa pode possuir vários bilhetes ao longo do tempo.

Entretanto, uma placa pode possuir no máximo um bilhete com estado `aberto` simultaneamente.

Exemplo válido:

```text
ABC1D23 → encerrado
ABC1D23 → cancelado
ABC1D23 → encerrado
ABC1D23 → aberto
```

O último bilhete só pode ser criado se não existir outro bilhete `aberto` para a mesma placa.

O encerramento ou cancelamento de um bilhete libera a placa para uma nova abertura.

---

## 9. Regra de Cobrança

A cobrança deve utilizar a variante:

* tarifa horária: `600` centavos;
* fração: `30` minutos;
* valor de cada fração: `300` centavos;
* teto por bilhete: `5000` centavos;
* tolerância: `0` minutos.

A duração deve ser calculada entre a entrada e a saída.

Para duração positiva, a quantidade de frações é determinada pelo arredondamento para cima da duração em minutos dividida por `30`.

Exemplos:

| Duração | Frações | Valor antes do teto |
| ------: | ------: | ------------------: |
|   1 min |       1 |                 300 |
|  29 min |       1 |                 300 |
|  30 min |       1 |                 300 |
|  31 min |       2 |                 600 |
|  59 min |       2 |                 600 |
|  60 min |       2 |                 600 |
|  61 min |       3 |                 900 |
|  90 min |       3 |                 900 |
|  91 min |       4 |                1200 |

Como `TOLERANCIA_MINUTOS = 0`, somente duração igual a zero possui valor gratuito. Qualquer duração positiva deve ser submetida ao cálculo normal da cobrança.

O valor final da cobrança não pode ultrapassar `5000` centavos por bilhete.

O teto é aplicado individualmente ao valor do bilhete.

---

## 10. Não Inventar Regras

Quando o contrato não definir explicitamente determinado comportamento, a implementação não deve inventar:

* novos endpoints;
* novos códigos de erro;
* novos estados;
* novos campos obrigatórios;
* novas regras de cobrança;
* novas transições de estado;
* formatos alternativos de resposta.

Decisões internas de implementação são permitidas somente quando não alterarem o comportamento externo definido pelo contrato.

---

## 11. Ordenação

Quando o contrato determinar que registros sejam retornados dos mais recentes para os mais antigos, a ordenação deve utilizar o critério temporal correspondente ao registro, priorizando o instante de entrada do bilhete quando aplicável.

Não assumir que o identificador numérico representa ordem cronológica sem que isso seja explicitamente definido.

---

## 12. Testabilidade

A solução deve permitir a verificação determinística das regras de negócio.

Especialmente:

* a abertura deve aceitar `entrada` quando o contrato permitir;
* o cálculo de cobrança deve depender dos instantes registrados;
* as regras de estado devem ser reproduzíveis;
* os casos de fronteira devem poder ser testados sem depender de espera real desnecessária.

---

## 13. Configuração de Execução

A aplicação deve estar acessível pela porta definida pela variante:

```text
PORTA_SERVICO=8005
```

A porta utilizada pela suíte de correção deve ser considerada parte do requisito de execução da prova.

Não substituir `PORTA_SERVICO` por uma porta fixa diferente.

---

## 14. Qualidade e Segurança

A solução deve:

* evitar credenciais e segredos hardcoded;
* não depender de serviços externos desnecessários;
* manter os dados necessários para os testes de forma reproduzível;
* possuir testes automatizados para as regras críticas;
* manter separação clara entre regras de domínio, contrato HTTP e infraestrutura;
* ser executável pela infraestrutura esperada pela prova.

---

## 15. Regra de Precedência

A ordem de referência para decisões é:

1. contrato funcional da prova;
2. requisitos explicitamente definidos no enunciado;
3. `constitution.md`;
4. `spec.md`;
5. `plan.md`;
6. `tests.md`;
7. `tasks.md`.

Nenhum documento posterior pode contradizer uma regra estabelecida pelo contrato.

Os documentos devem detalhar e organizar o comportamento, não modificar o contrato.
