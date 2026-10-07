# Especificação Funcional — Sistema de Gestão de Bilhetes

## 1. Objetivo

O sistema gerencia bilhetes de estacionamento associados a placas de veículos.

O sistema deve permitir:

* abrir bilhetes;
* encerrar bilhetes;
* cancelar bilhetes;
* listar bilhetes ativos;
* consultar histórico por placa;
* gerar relatório diário;
* calcular a cobrança conforme as regras da variante;
* impedir mais de um bilhete aberto simultaneamente para a mesma placa.

A implementação deve respeitar integralmente o contrato da API e as regras definidas neste documento.

---

# 2. Parâmetros da Variante

A prova utiliza os seguintes valores:

| Parâmetro              |    Valor |
| ---------------------- | -------: |
| `TARIFA_HORA_CENTAVOS` |    `600` |
| `FRACAO_MINUTOS`       |     `30` |
| `TETO_DIARIO_CENTAVOS` |   `5000` |
| `TOLERANCIA_MINUTOS`   |      `0` |
| `PORTA_SERVICO`        |   `8005` |
| Fuso horário           | `-03:00` |

A tarifa horária de `600` centavos corresponde a R$ 6,00.

Como a fração é de 30 minutos, cada fração possui valor de:

```text
600 / (60 / 30) = 300 centavos
```

Portanto, cada fração de 30 minutos custa R$ 3,00.

---

# 3. Modelo Conceitual do Bilhete

Cada bilhete possui, conforme o contrato, informações relacionadas a:

* identificador;
* placa;
* instante de entrada;
* instante de saída quando aplicável;
* estado;
* valor da cobrança quando aplicável.

Os estados possíveis são:

```text
aberto
encerrado
cancelado
```

Um bilhete é criado inicialmente no estado `aberto`.

---

# 4. Máquina de Estados

As transições válidas são:

| Estado atual | Operação | Estado resultante |
| ------------ | -------- | ----------------- |
| inexistente  | abrir    | `aberto`          |
| `aberto`     | encerrar | `encerrado`       |
| `aberto`     | cancelar | `cancelado`       |

Não deve existir retorno para `aberto` depois que o bilhete for encerrado ou cancelado.

As operações sobre estados inválidos devem utilizar somente os erros definidos pelo contrato.

Não deve ser criado um código de erro adicional para situações que não possuam código explicitamente definido.

---

# 5. Regras Globais

## 5.1 Placa

A placa deve:

* ser obrigatória;
* possuir exatamente 7 caracteres;
* conter somente caracteres alfanuméricos;
* utilizar caracteres maiúsculos.

Uma placa que não respeite essas condições deve resultar em:

```text
HTTP 422
erro: placa_invalida
```

A validação da placa deve ocorrer antes da verificação de conflitos de negócio.

---

## 5.2 Data e Hora

Datas e horários devem utilizar ISO-8601 com fuso horário.

O fuso utilizado pela aplicação é:

```text
-03:00
```

Quando `entrada` não for informada na abertura do bilhete, o sistema deve utilizar o instante atual.

Quando `entrada` for informada, o sistema deve utilizar o instante fornecido.

Uma entrada inválida deve resultar em:

```text
HTTP 422
erro: entrada_invalida
```

---

## 5.3 Validação antes das Regras de Negócio

As validações estruturais devem ocorrer antes das validações de estado ou conflito.

Exemplo:

Se já existir um bilhete aberto para determinada placa, mas uma nova requisição utilizar uma placa inválida, o resultado esperado é:

```text
HTTP 422
erro: placa_invalida
```

e não:

```text
HTTP 409
erro: bilhete_em_aberto
```

---

# 6. UC1 — Abrir Bilhete

## 6.1 Objetivo

Criar um novo bilhete para uma placa.

## 6.2 Endpoint

```text
POST /bilhetes
```

## 6.3 Entrada

O request deve aceitar:

* `placa` — obrigatório;
* `entrada` — opcional.

Quando `entrada` estiver ausente, utilizar o instante atual.

Quando `entrada` estiver presente, utilizar o instante informado.

## 6.4 Validações

A placa deve ser validada conforme as regras globais.

A entrada, quando fornecida, deve ser um instante ISO-8601 válido com fuso.

A validação deve ocorrer antes da verificação de bilhete aberto.

## 6.5 Regra de conflito

Uma placa pode possuir no máximo um bilhete no estado `aberto`.

Se já existir um bilhete aberto para a placa, a abertura deve falhar com:

```text
HTTP 409
erro: bilhete_em_aberto
```

Bilhetes `encerrados` ou `cancelados` não impedem a abertura de um novo bilhete para a mesma placa.

## 6.6 Sucesso

Quando os dados forem válidos e não existir conflito, o sistema deve responder:

```text
HTTP 201
```

O bilhete criado deve conter:

* `id`;
* `placa`;
* `entrada`;
* `status` com valor `aberto`.

## 6.7 Critérios de Aceite

**CA-UC1-01**

Dado uma placa válida sem bilhete aberto, ao realizar `POST /bilhetes`, o sistema deve criar um novo bilhete e retornar HTTP 201.

**CA-UC1-02**

O bilhete recém-criado deve possuir status `aberto`.

**CA-UC1-03**

Quando `entrada` não for informada, o sistema deve registrar o instante atual.

**CA-UC1-04**

Quando `entrada` for informada e válida, o sistema deve preservar esse instante.

**CA-UC1-05**

Uma placa inválida deve resultar em HTTP 422 com `placa_invalida`.

**CA-UC1-06**

Uma entrada inválida deve resultar em HTTP 422 com `entrada_invalida`.

**CA-UC1-07**

Uma segunda abertura para uma placa que já possui bilhete aberto deve resultar em HTTP 409 com `bilhete_em_aberto`.

**CA-UC1-08**

Depois que o bilhete anterior for encerrado ou cancelado, a mesma placa poderá abrir um novo bilhete.

---

# 7. UC2 — Encerrar Bilhete

## 7.1 Objetivo

Encerrar um bilhete aberto e calcular seu valor de cobrança.

## 7.2 Endpoint

```text
POST /bilhetes/{id}/encerrar
```

## 7.3 Bilhete inexistente

Caso o identificador informado não corresponda a um bilhete existente:

```text
HTTP 404
erro: bilhete_nao_encontrado
```

## 7.4 Bilhete já encerrado

Caso o bilhete já esteja encerrado:

```text
HTTP 409
erro: bilhete_ja_encerrado
```

## 7.5 Estado

Somente bilhetes `abertos` podem ser encerrados.

Ao realizar o encerramento:

```text
aberto → encerrado
```

O instante de saída deve corresponder ao momento do encerramento.

## 7.6 Cálculo da duração

A duração deve ser obtida pela diferença entre:

```text
saída - entrada
```

A duração utilizada para cobrança deve ser considerada em minutos conforme as regras do contrato.

---

# 8. Regra de Cobrança

## 8.1 Tolerância

A variante possui:

```text
TOLERANCIA_MINUTOS = 0
```

Portanto:

* duração igual a 0 minuto → valor zero;
* qualquer duração positiva → cobrança normal.

Não existe período gratuito positivo nesta variante.

A tolerância não deve ser subtraída da duração.

---

## 8.2 Quantidade de Frações

A fração da variante é:

```text
FRACAO_MINUTOS = 30
```

A quantidade de frações deve ser arredondada para cima.

A regra é:

```text
frações = teto(duração_em_minutos / 30)
```

Exemplos:

| Duração | Frações |
| ------: | ------: |
|       0 |       0 |
|       1 |       1 |
|      29 |       1 |
|      30 |       1 |
|      31 |       2 |
|      59 |       2 |
|      60 |       2 |
|      61 |       3 |
|      89 |       3 |
|      90 |       3 |
|      91 |       4 |

---

## 8.3 Valor da Fração

A tarifa horária é:

```text
600 centavos
```

Uma hora possui duas frações de 30 minutos.

Portanto:

```text
valor_da_fração = 600 / 2
valor_da_fração = 300 centavos
```

Cada fração corresponde a R$ 3,00.

---

## 8.4 Valor Bruto

O valor antes da aplicação do teto é:

```text
valor_bruto = quantidade_de_frações × 300
```

Exemplos:

| Duração | Frações | Valor bruto |
| ------: | ------: | ----------: |
|   1 min |       1 |         300 |
|  30 min |       1 |         300 |
|  31 min |       2 |         600 |
|  60 min |       2 |         600 |
|  61 min |       3 |         900 |
|  90 min |       3 |         900 |
| 120 min |       4 |        1200 |

Todos os valores são expressos em centavos.

---

## 8.5 Teto

O teto da variante é:

```text
TETO_DIARIO_CENTAVOS = 5000
```

O valor final de um bilhete nunca pode ultrapassar 5000 centavos.

Conceitualmente:

```text
valor_final = mínimo(valor_bruto, 5000)
```

Como cada fração vale 300 centavos:

* 16 frações = 4800 centavos;
* 17 frações = 5100 centavos;
* 17 ou mais frações devem resultar em 5000 centavos.

O teto é aplicado individualmente ao bilhete.

---

## 8.6 Critérios de Aceite da Cobrança

**CA-COB-01**

Uma duração positiva de até 30 minutos deve gerar uma fração e cobrança de 300 centavos.

**CA-COB-02**

Uma duração de 31 minutos deve gerar duas frações e cobrança de 600 centavos.

**CA-COB-03**

Uma duração de 60 minutos deve gerar duas frações e cobrança de 600 centavos.

**CA-COB-04**

Uma duração de 61 minutos deve gerar três frações e cobrança de 900 centavos.

**CA-COB-05**

Uma duração de 16 frações deve gerar 4800 centavos.

**CA-COB-06**

Uma duração de 17 frações deve ser limitada a 5000 centavos.

**CA-COB-07**

Durações superiores ao limite de 17 frações continuam limitadas a 5000 centavos.

**CA-COB-08**

Nenhum cálculo monetário deve utilizar representação em ponto flutuante.

---

# 9. UC3 — Listar Bilhetes Ativos

## 9.1 Objetivo

Consultar os bilhetes atualmente abertos.

## 9.2 Regra

A consulta deve retornar somente bilhetes cujo estado seja:

```text
aberto
```

Bilhetes `encerrados` e `cancelados` não devem aparecer na lista de ativos.

## 9.3 Ordenação

Os bilhetes ativos devem ser retornados dos mais recentes para os mais antigos, considerando o instante de entrada.

A implementação não deve assumir que o `id` representa necessariamente a ordem cronológica.

## 9.4 Critérios de Aceite

**CA-UC3-01**

Um bilhete aberto deve aparecer na listagem.

**CA-UC3-02**

Um bilhete encerrado não deve aparecer na listagem.

**CA-UC3-03**

Um bilhete cancelado não deve aparecer na listagem.

**CA-UC3-04**

Quando houver múltiplos bilhetes abertos, o mais recente deve aparecer antes do mais antigo.

---

# 10. UC4 — Relatório Diário

## 10.1 Objetivo

Consultar os dados consolidados de determinado dia.

O relatório deve retornar informações referentes à data solicitada pelo contrato.

O campo `data` deve representar a data consultada.

## 10.2 Campos

O relatório possui os seguintes campos:

```text
data
total_bilhetes
faturamento_centavos
tempo_medio_minutos
```

## 10.3 Faturamento

O faturamento deve utilizar valores em centavos.

Bilhetes cancelados não devem gerar cobrança.

Os valores considerados no faturamento devem seguir a definição de bilhetes do dia estabelecida pelo contrato.

## 10.4 Tempo Médio

O `tempo_medio_minutos` deve considerar somente bilhetes encerrados no dia consultado.

Bilhetes:

* ainda abertos;
* cancelados;
* encerrados em outro dia;

não devem contribuir para essa média.

## 10.5 Arredondamento da Média

A média deve ser arredondada para o inteiro mais próximo utilizando a regra de metade para cima.

Exemplos:

```text
10,4 → 10
10,5 → 11
10,6 → 11
```

Não utilizar truncamento simples.

## 10.6 Critérios de Aceite

**CA-UC4-01**

O relatório deve identificar corretamente a data consultada.

**CA-UC4-02**

O faturamento deve ser expresso em centavos.

**CA-UC4-03**

Bilhetes cancelados não devem gerar cobrança.

**CA-UC4-04**

Somente bilhetes encerrados no dia devem contribuir para o tempo médio.

**CA-UC4-05**

Uma média terminando em `.5` deve ser arredondada para cima.

**CA-UC4-06**

Bilhetes ainda abertos não devem contribuir para o tempo médio.

---

# 11. UC5 — Cancelar Bilhete

## 11.1 Objetivo

Cancelar um bilhete que ainda esteja aberto.

## 11.2 Regra

O cancelamento somente pode ocorrer para um bilhete no estado:

```text
aberto
```

A transição é:

```text
aberto → cancelado
```

## 11.3 Bilhete inexistente

Quando o bilhete não existir:

```text
HTTP 404
erro: bilhete_nao_encontrado
```

## 11.4 Bilhete que não está aberto

Quando o bilhete existente não estiver aberto:

```text
HTTP 409
erro: bilhete_nao_aberto
```

Isso se aplica aos estados definidos pelo contrato como não abertos.

## 11.5 Cobrança

O cancelamento não deve gerar cobrança.

Um bilhete cancelado não deve ser tratado como bilhete encerrado para fins de cobrança ou tempo médio de permanência.

## 11.6 Critérios de Aceite

**CA-UC5-01**

Um bilhete aberto pode ser cancelado.

**CA-UC5-02**

Depois do cancelamento, o status deve ser `cancelado`.

**CA-UC5-03**

O cancelamento de um bilhete inexistente deve retornar 404.

**CA-UC5-04**

O cancelamento de um bilhete que não esteja aberto deve retornar 409 com `bilhete_nao_aberto`.

**CA-UC5-05**

Um bilhete cancelado não deve gerar cobrança.

---

# 12. UC6 — Histórico por Placa

## 12.1 Objetivo

Consultar o histórico de bilhetes associados a uma placa.

## 12.2 Regra

O histórico deve considerar os bilhetes da placa independentemente do estado:

* `aberto`;
* `encerrado`;
* `cancelado`.

## 12.3 Ordenação

Os registros devem ser retornados dos mais recentes para os mais antigos, utilizando o instante de entrada como referência temporal.

## 12.4 Placa sem Histórico

Quando não houver bilhetes associados à placa consultada, o resultado deve ser uma coleção vazia conforme o contrato.

A ausência de histórico não deve ser tratada como erro quando o contrato determinar retorno vazio.

## 12.5 Critérios de Aceite

**CA-UC6-01**

Um bilhete aberto deve aparecer no histórico.

**CA-UC6-02**

Um bilhete encerrado deve aparecer no histórico.

**CA-UC6-03**

Um bilhete cancelado deve aparecer no histórico.

**CA-UC6-04**

O histórico deve apresentar os registros mais recentes primeiro.

**CA-UC6-05**

Uma placa sem histórico deve retornar uma coleção vazia.

---

# 13. UC7 — Tolerância Gratuita

## 13.1 Regra da Variante

A variante possui:

```text
TOLERANCIA_MINUTOS = 0
```

Portanto, não existe tolerância gratuita para qualquer duração positiva.

Apenas uma duração de exatamente zero minutos possui cobrança igual a zero.

## 13.2 Critérios de Aceite

**CA-UC7-01**

Duração de 0 minuto deve resultar em cobrança de 0 centavos.

**CA-UC7-02**

Duração de 1 minuto deve resultar em cobrança de 300 centavos.

**CA-UC7-03**

A aplicação não deve descontar minutos de tolerância da duração antes de calcular as frações.

---

# 14. UC8 — Uma Vaga por Placa

## 14.1 Regra

Uma placa pode possuir no máximo um bilhete aberto simultaneamente.

A existência de bilhetes encerrados ou cancelados não impede uma nova abertura.

Exemplo:

```text
ABC1D23 → encerrado
ABC1D23 → cancelado
ABC1D23 → encerrado
ABC1D23 → novo bilhete aberto
```

é permitido.

Por outro lado:

```text
ABC1D23 → aberto
ABC1D23 → novo bilhete
```

deve resultar em conflito.

## 14.2 Critérios de Aceite

**CA-UC8-01**

Uma placa sem bilhete aberto pode abrir um novo bilhete.

**CA-UC8-02**

Uma placa com bilhete aberto não pode abrir outro simultaneamente.

**CA-UC8-03**

Depois do encerramento do bilhete anterior, a placa pode abrir outro.

**CA-UC8-04**

Depois do cancelamento do bilhete anterior, a placa pode abrir outro.

---

# 15. Contrato de Erros

Os erros devem utilizar exatamente os códigos definidos pelo contrato.

| Situação                                 | HTTP | Erro                     |
| ---------------------------------------- | ---: | ------------------------ |
| Placa inválida                           |  422 | `placa_invalida`         |
| Entrada inválida                         |  422 | `entrada_invalida`       |
| Bilhete aberto já existente para a placa |  409 | `bilhete_em_aberto`      |
| Bilhete inexistente ao encerrar          |  404 | `bilhete_nao_encontrado` |
| Bilhete já encerrado                     |  409 | `bilhete_ja_encerrado`   |
| Bilhete inexistente ao cancelar          |  404 | `bilhete_nao_encontrado` |
| Bilhete não aberto ao cancelar           |  409 | `bilhete_nao_aberto`     |

Não criar novos identificadores de erro para situações que não estejam definidos pelo contrato.

---

# 16. Identificador do Bilhete

Cada bilhete deve possuir um identificador único.

O identificador deve ser retornado no campo `id` conforme o contrato.

A especificação não impõe uma estratégia interna específica para geração do identificador, desde que:

* seja único;
* possa identificar o bilhete posteriormente;
* seja compatível com o formato esperado pelo contrato.

Não assumir que o identificador deve ser UUID ou outra tecnologia específica sem exigência contratual.

---

# 17. Regras de Ordenação

Sempre que uma consulta determinar "mais recentes primeiro":

1. considerar o instante temporal definido para o registro;
2. ordenar do instante mais recente para o mais antigo;
3. não utilizar o `id` como substituto automático do instante temporal.

Para ativos e histórico, o instante de entrada é a referência temporal principal.

---

# 18. Critérios Gerais de Aceite

A implementação será considerada funcionalmente consistente quando:

1. todos os endpoints do contrato forem disponibilizados;
2. os métodos HTTP estiverem corretos;
3. os campos de request e response respeitarem o contrato;
4. os códigos HTTP estiverem corretos;
5. os erros utilizarem os identificadores definidos;
6. as validações ocorrerem antes dos conflitos de negócio;
7. a máquina de estados for respeitada;
8. uma placa nunca possuir dois bilhetes abertos simultaneamente;
9. o cálculo utilizar frações de 30 minutos;
10. qualquer duração positiva gerar pelo menos uma fração;
11. cada fração representar 300 centavos;
12. o teto por bilhete for de 5000 centavos;
13. o cancelamento não gerar cobrança;
14. o relatório calcular a média somente sobre encerramentos do dia;
15. a média utilizar arredondamento de metade para cima;
16. ativos e histórico forem ordenados dos mais recentes para os mais antigos;
17. valores monetários forem tratados como inteiros em centavos;
18. a aplicação estiver acessível pela porta `8005`.
