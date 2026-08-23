# Orcamento-AI_-API_Inteligente_com_Spring_AI
 > Desafio de Projeto da trilha [Spring Boot Learning Track](https://github.com/digitalinnovationone/dio-spring-boot-learning-track)> módulo `05-spring-ai`
O que o projeto faz

API que recebe um **comando de voz** (áudio), transcreve o áudio em texto, usa um modelo de IA
para entender a intenção do usuário e executa uma ação real da aplicação — registrar uma
transação financeira ou consultar o saldo — usando **Tool Calling**. Ao final, gera também uma
resposta em áudio (texto para voz).

Fluxo principal:
Áudio (upload) --> Transcrição (Whisper) --> ChatClient + Tool Calling --> Ação real (JPA)
|
v
Resposta em texto + áudio (TTS)


Também expõe endpoints REST tradicionais para cadastrar e consultar transações diretamente,
sem passar pelo fluxo de voz — úteis para testar a lógica de negócio isoladamente.

Tecnologias usadas

- Java 17
- Spring Boot 3.3
- Spring AI 1.0 (`ChatClient`, Tool Calling, transcrição de áudio e geração de voz via OpenAI)
- Spring Data JPA + H2 (banco em memória, só para facilitar testes/demonstração)
- Bean Validation (`jakarta.validation`)
- JUnit 5 + Mockito

Estrutura do projeto
src/main/java/com/dio/orcamentoai/
├── config/ -> ChatClientConfig (prompt de sistema + registro das tools)
├── domain/ -> Transacao (entidade JPA), TipoTransacao
├── repository/ -> TransacaoRepository
├── dto/ -> Requests/Responses com validação
├── service/ -> TransacaoService (regras de negócio) e ComandoDeVozService (orquestra o fluxo de voz)
├── tools/ -> FinanceiroTools (funções expostas para a IA via @Tool)
├── controller/ -> TransacaoController (REST) e ComandoDeVozController (upload de áudio)
└── exception/ -> GlobalExceptionHandler (validações e erros de negócio)


## Como executar

1. Exporte sua chave da OpenAI (necessária para transcrição, chat e geração de voz):

```bash
   export OPENAI_API_KEY=sua-chave-aqui
```

2. Rode a aplicação:

```bash
   mvn spring-boot:run
```

3. A API sobe em `http://localhost:8080`. O console do H2 fica em `/h2-console`
   (JDBC URL: `jdbc:h2:mem:orcamento`).

> **Nota:** este projeto foi desenvolvido/organizado com apoio de IA para acelerar a escrita do
> código; antes de considerar pronto, rode `mvn clean verify` na sua máquina, revise as versões
> das dependências do Spring AI no `pom.xml` (podem ter evoluído) e ajuste o que for necessário.

Como testar o fluxo principal

Via comando de voz** (envie um áudio, ex. `.mp3` ou `.wav`, dizendo algo como
"gastei 50 reais no mercado hoje"):

bash
curl -X POST http://localhost:8080/api/comandos-de-voz \
  -F "audio=@comando.mp3"


**Via REST direto** (sem passar pela IA):

bash
registrar uma transação
curl -X POST http://localhost:8080/api/transacoes \
  -H "Content-Type: application/json" \
  -d '{"descricao":"Mercado do mês","valor":250.00,"tipo":"DESPESA","categoria":"Mercado"}'

consultar saldo
curl http://localhost:8080/api/transacoes/saldo

consultar saldo agrupado por categoria
curl http://localhost:8080/api/transacoes/saldo-por-categoria

consultar por categoria
curl http://localhost:8080/api/transacoes/categoria/Mercado

# consultar por período
curl "http://localhost:8080/api/transacoes/periodo?inicio=2026-08-01&fim=2026-08-31"


Testes automatizados:

bash
mvn test


Os testes em `TransacaoServiceTest` cobrem: registro de transação válida, rejeição de valor
zero/negativo, rejeição de categoria vazia, cálculo de saldo geral e cálculo de saldo por
categoria.

Melhoria implementada em relação ao projeto base

Em vez de apenas replicar o projeto apresentado nas aulas, evoluí três pontos:

1. Nova consulta financeira** — `consultarSaldoPorCategoria` (tool) e o endpoint
   `GET /api/transacoes/saldo-por-categoria`, que agrupam o saldo por categoria. Isso permite
   perguntas como "quanto eu já gastei com Mercado esse mês?" em vez de só o saldo total.
2. Validações antes de salvar uma transação** — valor precisa ser maior que zero e categoria
   é obrigatória, tanto na camada REST (`@Valid` + Bean Validation) quanto na camada de serviço
   (usada também pelo Tool Calling da IA, garantindo que a IA não consiga burlar a regra).
3. Tratamento de erros centralizado** — `GlobalExceptionHandler` retorna respostas HTTP 400
   claras e estruturadas em vez de stack traces, tanto para erros de validação quanto de regra
   de negócio.

Diferenças em relação à arquitetura oficial da trilha

O projeto de referência (`05-spring-ai` do
[dio-spring-boot-learning-track](https://github.com/digitalinnovationone/dio-spring-boot-learning-track))
segue uma arquitetura em camadas DDD/Clean Architecture, usada em todos os módulos da trilha:

domain/ -> modelo de negócio, regras, contratos de repositório
application/ -> use cases (uma classe por ação de negócio)
infrastructure/ -> adapters (REST, JPA, integrações externas)


Este projeto optou por uma organização por **camada técnica** (`domain`, `repository`, `dto`,
`service`, `tools`, `controller`, `exception`) em vez de por camada DDD. Principais diferenças:

| Ponto | Este projeto | Padrão oficial da trilha |
|---|---|---|
| Build tool | Maven | Gradle (`./gradlew`) |
| Pacote base | `com.dio.orcamentoai` | `dio.budgeting` |
| Lógica de negócio | `TransacaoService` único com vários métodos | um use case por ação (`RegistrarTransacaoUseCase`, `ConsultarSaldoUseCase`, etc.) |
| Repositório | `TransacaoRepository extends JpaRepository` direto | contrato no domínio (`domain/`) + implementação JPA separada em `infrastructure/` |
| IDs | `Long id` gerado pelo JPA | ID fortemente tipado (ex: `TransactionId`) |
| Modelos do Spring AI | `OpenAiAudioTranscriptionModel`, `OpenAiAudioSpeechModel` (específicos da OpenAI) | interfaces genéricas do Spring AI (`TranscriptionModel`, `TextToSpeechModel`) |
| Modelagem `class` vs `record` | `Transacao` como classe, DTOs como `record` | mesma convenção (compatível) |

Optei por manter a estrutura por camada técnica porque é mais direta para um projeto deste
tamanho, mas reconheço que a separação `domain`/`application`/`infrastructure` do projeto oficial
deixa mais explícito que a regra de negócio (use case) não depende de detalhes de framework —
por exemplo, trocar o banco H2 por Postgres, ou trocar o provedor de IA, exigiria mexer só na
camada `infrastructure`, sem tocar nas regras de negócio.

## O que aprendi

Esse desafio me fez entender na prática como conectar IA com uma aplicação de verdade, não só mandar um prompt e receber texto de volta. A parte que mais me chamou atençãofoi o Tool Calling:
eu só escrevo o método (registrar transação, consultar saldo, etc.) com uma descrição boa em `@Tool`, e o próprio `ChatClient` decide sozinho qual ferramenta chamar de acordo com o que a pessoa falou. Não precisei escrever nenhum "if o usuário disse X, chama a função Y" — a IA que interpreta isso.

Outra coisa que ficou clara foi a diferença entre validar na entrada (o `@Valid` lá no controller, pegando erro de payload) e validar na regra de negócio (dentro do service). Como o Tool Calling chama o service direto, se eu só validasse no controller, a IA conseguiria burlar essa validação registrando qualquer coisa. Colocando a regra no service também, garanti que nem o fluxo de voz nem o REST conseguem passar por cima disso.

Também entendi na prática por que faz sentido separar cada etapa em uma classe/serviço diferente: transcrever áudio, mandar pra IA e gerar a resposta em voz são responsabilidades bem distintas, e deixando isso separado ficou muito mais fácil de testar cada pedaço isolado (dá pra testar a lógica de registrar transação sem precisar de áudio nenhum, por exemplo).

Por fim, comparar meu projeto com a estrutura oficial da trilha (DDD com domain/application/ infrastructure) me fez pensar em trade-off de arquitetura: minha organização por camada técnica é mais rápida de montar num projeto desse tamanho, mas eu vi na prática por que separar use case de detalhe de infraestrutura ajuda muito quando você imagina trocar peça (banco, provedor de IA) sem mexer na regra de negócio.

