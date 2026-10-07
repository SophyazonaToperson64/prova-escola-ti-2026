# Plano de Implementação — Sistema de Gestão de Bilhetes

## 1. Objetivo

Este documento define a estratégia técnica para transformar a especificação funcional em uma implementação executável e testável.

O plano deve orientar a implementação sem alterar o comportamento definido no contrato ou no `spec.md`.

As decisões internas podem ser adaptadas conforme a estrutura do projeto, desde que os contratos externos, regras de negócio e critérios de aceite permaneçam inalterados.

---

# 2. Princípios de Implementação

A implementação deve priorizar:

1. aderência ao contrato;
2. simplicidade;
3. testabilidade;
4. determinismo;
5. separação das responsabilidades;
6. facilidade de execução pela infraestrutura da prova.

Não adicionar tecnologias ou componentes que não contribuam diretamente para esses objetivos.

---

# 3. Organização Arquitetural

A solução deve manter separação entre as seguintes responsabilidades:

```text id="7j8t9r"
HTTP / API
    ↓
Validação de entrada
    ↓
Serviço / Regras de negócio
    ↓
Persistência
```

### Camada HTTP

Responsável por:

* receber requests;
* validar o formato conforme o contrato;
* encaminhar dados para as regras de negócio;
* converter resultados para responses HTTP;
* retornar os códigos HTTP definidos no contrato.

A camada HTTP não deve conter toda a lógica de cobrança ou das transições de estado.

### Camada de domínio/serviço

Responsável por:

* regras de estado;
* abertura;
* encerramento;
* cancelamento;
* unicidade de bilhete aberto por placa;
* cálculo de cobrança;
* regras do relatório;
* histórico;
* ordenação.

### Persistência

Responsável por:

* armazenar bilhetes;
* localizar bilhete por `id`;
* localizar bilhetes por placa;
* localizar bilhetes abertos;
* atualizar estado e informações de encerramento;
* fornecer dados necessários aos relatórios.

A tecnologia de persistência pode ser escolhida de acordo com a estrutura da solução, desde que cumpra esses requisitos.

---

# 4. Modelo de Domínio

O conceito central da aplicação é o `Bilhete`.

O domínio deve representar, no mínimo, as informações necessárias pelo contrato:

* `id`;
* `placa`;
* `entrada`;
* estado;
* informações de saída quando o bilhete for encerrado;
* valor da cobrança quando aplicável.

Os estados devem ser representados de forma restrita aos valores:

```text id="y4v6gq"
aberto
encerrado
cancelado
```

A transição de estado deve ocorrer somente por operações previstas na especificação.

---

# 5. Validação

As validações devem ser executadas antes das regras de conflito e estado.

A estratégia deve separar:

### Validação estrutural

Verificar:

* presença dos campos obrigatórios;
* formato da placa;
* quantidade de caracteres;
* caracteres permitidos;
* formato de data/hora;
* demais restrições estruturais previstas no contrato.

### Validação de negócio

Somente depois da validação estrutural, verificar:

* existência do bilhete;
* estado do bilhete;
* existência de outro bilhete aberto para a mesma placa;
* demais conflitos definidos pelo contrato.

Essa ordem é obrigatória para garantir que uma entrada inválida produza `422` antes de um possível `409`.

---

# 6. Tratamento de Data e Hora

A aplicação deve utilizar timestamps com o fuso `-03:00`.

A abertura deve possuir comportamento determinístico:

* `entrada` fornecida → utilizar o valor fornecido;
* `entrada` omitida → utilizar o instante atual.

O cálculo de duração deve utilizar os timestamps armazenados, e não estimativas baseadas em identificadores ou ordem de inserção.

O código deve evitar lógica espalhada de manipulação temporal. Sempre que possível, concentrar cálculos de duração em uma responsabilidade de domínio/serviço.

---

# 7. Estratégia de Cobrança

O cálculo deve ser isolado em uma responsabilidade específica para facilitar testes.

Parâmetros da variante:

```text id="xq1c6a"
TARIFA_HORA_CENTAVOS = 600
FRACAO_MINUTOS = 30
TETO_DIARIO_CENTAVOS = 5000
TOLERANCIA_MINUTOS = 0
```

A implementação deve seguir esta sequência conceitual:

```text id="px4m6d"
entrada
   ↓
saída
   ↓
duração em minutos
   ↓
verificação da tolerância
   ↓
quantidade de frações
   ↓
valor da fração
   ↓
valor bruto
   ↓
aplicação do teto
   ↓
valor final em centavos
```

Para esta variante:

```text id="y5t9cc"
valor_da_fração = 300 centavos
```

A quantidade de frações deve utilizar arredondamento para cima.

O teto deve ser aplicado somente depois do cálculo do valor bruto.

---

# 8. Representação Monetária

Todos os valores monetários devem ser armazenados e processados como inteiros em centavos.

Não utilizar `float` ou equivalente para representar dinheiro.

Exemplo:

```text id="f7c1w0"
R$ 3,00 → 300
R$ 6,00 → 600
R$ 50,00 → 5000
```

Essa decisão deve ser aplicada tanto ao domínio quanto às respostas da API.

---

# 9. Controle dos Estados

As transições devem ser centralizadas na lógica de domínio.

Fluxo esperado:

```text id="0t8x6r"
             abrir
                ↓
             ABERTO
             /    \
            /      \
       encerrar    cancelar
          ↓           ↓
     ENCERRADO    CANCELADO
```

Evitar permitir alterações arbitrárias do estado diretamente pela camada HTTP ou persistência.

A regra de que somente `aberto` pode ser encerrado ou cancelado deve ser aplicada no domínio.

---

# 10. Unicidade por Placa

A criação de um bilhete deve verificar se existe outro bilhete com:

```text id="v7m3sp"
mesma placa
+
status = aberto
```

Se existir, retornar o conflito definido pelo contrato.

A consulta não deve considerar bilhetes encerrados ou cancelados como bloqueadores de uma nova abertura.

Quando um bilhete aberto for encerrado ou cancelado, a placa poderá possuir outro bilhete aberto posteriormente.

---

# 11. Listagem de Ativos

A consulta de ativos deve:

1. buscar somente bilhetes `abertos`;
2. ordenar pelo instante de entrada em ordem decrescente;
3. retornar o formato definido pelo contrato.

Não utilizar o `id` como critério de ordenação quando o instante de entrada estiver disponível.

---

# 12. Histórico

A consulta por placa deve:

1. localizar todos os bilhetes associados à placa;
2. incluir os estados permitidos pelo contrato;
3. ordenar pelo instante de entrada em ordem decrescente;
4. retornar coleção vazia quando não houver registros, se esse for o comportamento definido pelo contrato.

O histórico não deve ser confundido com a listagem de ativos.

---

# 13. Relatório Diário

A geração do relatório deve separar claramente:

* seleção dos registros pertencentes à data;
* cálculo do faturamento;
* cálculo da duração;
* cálculo da média;
* arredondamento da média.

O tempo médio deve considerar somente bilhetes encerrados no dia consultado.

A média deve utilizar arredondamento de metade para cima.

Não utilizar arredondamento bancário ou truncamento quando a regra exigir que `x,5` seja arredondado para o inteiro superior.

---

# 14. Identificadores

O sistema deve gerar identificadores únicos para os bilhetes.

A estratégia interna de geração do identificador não deve ser especificada além do necessário para cumprir o contrato.

Não introduzir UUID, sequência específica ou outro mecanismo como requisito externo caso isso não seja exigido pelo contrato.

---

# 15. Tratamento de Erros

Centralizar o tratamento das respostas de erro para evitar divergências entre endpoints.

Os códigos definidos no contrato devem ser mapeados para os respectivos status HTTP:

```text id="d1d4b7"
422 → validações
404 → recurso inexistente
409 → conflitos de negócio/estado
```

Os identificadores de erro devem permanecer exatamente como definidos no contrato.

Não expor stack traces, exceções internas ou detalhes de infraestrutura como resposta pública da API.

---

# 16. Persistência

A persistência deve suportar pelo menos as seguintes operações:

### Bilhete

* criar;
* buscar por `id`;
* atualizar;
* listar por placa;
* localizar bilhete aberto por placa;
* consultar registros necessários para o relatório.

A escolha entre banco relacional, armazenamento em memória ou outra alternativa deve considerar o ambiente de correção e os requisitos do projeto.

A solução escolhida deve ser determinística e simples de executar.

Caso a infraestrutura da prova exija persistência entre requisições, o mecanismo utilizado deve manter os dados durante o ciclo de execução da aplicação.

---

# 17. Testabilidade

As regras de negócio devem ser estruturadas para permitir testes independentes da camada HTTP quando possível.

Priorizar testes unitários para:

* validação de placa;
* validação de entrada;
* cálculo de duração;
* cálculo de frações;
* cálculo monetário;
* tolerância;
* teto;
* transições de estado;
* conflito por placa;
* arredondamento da média.

Testes de integração devem verificar:

* endpoints;
* status HTTP;
* payloads;
* persistência;
* fluxo completo dos casos de uso.

---

# 18. Casos de Fronteira Obrigatórios

A implementação deve ser validada principalmente nas fronteiras das regras.

### Placa

* 6 caracteres;
* exatamente 7 caracteres;
* 8 caracteres;
* caracteres minúsculos;
* caracteres especiais;
* ausência;
* vazio.

### Fração

* 1 minuto;
* 29 minutos;
* 30 minutos;
* 31 minutos;
* 59 minutos;
* 60 minutos;
* 61 minutos.

### Teto

* valor abaixo de 5000;
* valor exatamente 5000;
* valor acima de 5000.

### Estados

* abrir;
* abrir novamente com placa ocupada;
* encerrar;
* encerrar novamente;
* cancelar;
* cancelar novamente;
* abrir após encerramento;
* abrir após cancelamento.

### Relatório

* nenhum encerramento;
* um encerramento;
* múltiplos encerramentos;
* média inteira;
* média terminando em `.5`;
* bilhete aberto;
* bilhete cancelado;
* encerramento em data diferente.

---

# 19. Configuração da Aplicação

A aplicação deve respeitar:

```text id="v4f2pr"
PORTA_SERVICO = 8005
```

A configuração deve permitir que a infraestrutura da prova consiga iniciar e acessar o serviço nessa porta.

Não assumir que a porta padrão de desenvolvimento seja suficiente.

Se houver diferença entre porta interna e porta exposta pelo ambiente, a configuração do serviço deve respeitar a porta exigida pela suíte de correção.

---

# 20. Containerização

A solução deve possuir uma forma reprodutível de execução em ambiente limpo.

A containerização deve:

* instalar as dependências necessárias;
* construir a aplicação;
* iniciar o serviço;
* expor a porta necessária;
* não depender de arquivos presentes apenas no ambiente do desenvolvedor.

O processo de build deve ser determinístico sempre que possível.

Não adicionar serviços externos desnecessários.

---

# 21. Manifesto de Dependências

Todas as dependências necessárias para compilar, executar e testar o projeto devem estar declaradas no mecanismo de gerenciamento correspondente à tecnologia escolhida.

Não depender de bibliotecas instaladas manualmente no ambiente.

As versões devem ser compatíveis entre si e com o ambiente da prova.

---

# 22. Testes Automatizados

A implementação deve possuir testes automatizados suficientes para demonstrar as regras críticas.

Prioridade:

1. contrato HTTP;
2. validações;
3. estados;
4. cobrança;
5. teto;
6. unicidade por placa;
7. relatório;
8. histórico;
9. ordenação.

Os testes devem cobrir tanto casos válidos quanto inválidos e casos de fronteira.

---

# 23. README e Execução

O projeto deve conter instruções objetivas para:

* instalar dependências;
* executar a aplicação;
* executar os testes;
* executar via container quando aplicável;
* identificar a porta utilizada.

O README deve refletir a implementação real e não conter instruções incompatíveis com o projeto.

---

# 24. Verificação Final

Antes de considerar a implementação concluída, realizar uma revisão contra:

1. contrato da API;
2. `constitution.md`;
3. `spec.md`;
4. `tests.md`;
5. `tasks.md`.

A revisão deve verificar especialmente:

* endpoints;
* métodos;
* campos;
* status HTTP;
* mensagens de erro;
* estados;
* cobrança;
* arredondamento;
* teto;
* relatório;
* ordenação;
* porta `8005`;
* execução por container;
* testes automatizados.

Nenhuma decisão técnica pode alterar um comportamento definido no contrato.
