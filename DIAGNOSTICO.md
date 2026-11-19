## Projeto Final — Clean Code
### Diagnóstico Inicial e Plano de Refatoração do Pisciclean (Backend)

**Projeto:** Pisciclean — API de Assistência para Manutenção e Limpeza de Piscinas  
**Integrantes:** Patricia Morais, Hian Rick e Katleen Martins  
**Repositório do frontend:** https://github.com/patriciamrs/projeto-psciclean  
**Repositório do backend:** https://github.com/Katleenmm/backend-piscina  
**Versão original (branch `original`):** commit `6c43824b5ad28168fea1ef32f2d9c9f144392468` (03/12/2025 — "Update README.md")

### 1. Descrição do software
O Pisciclean é um aplicativo mobile para auxiliar proprietários de piscinas nos cuidados e na manutenção da água. Este trabalho trata do **backend** do sistema: uma API REST em Spring Boot responsável pelo cadastro e autenticação de clientes, pelo registro e acompanhamento de solicitações de serviços presenciais e pelo armazenamento do histórico de conversas do chatbot, que é usado para gerar um resumo anexado à solicitação de serviço.

#### 1.1 Principais funcionalidades

- **Cadastro e autenticação:** criação de contas e login de clientes (`POST /cliente`, `POST /cliente/login`).
- **Gerenciamento de conta:** consulta, edição e remoção de clientes (`GET/PUT/DELETE /cliente/{id}`).
- **Solicitação de serviços:** criação de solicitações de atendimento presencial, com resumo opcional da conversa com o chatbot (`POST /servicos`).
- **Consulta de solicitações:** listagem geral (painel administrativo) e por cliente (`GET /servicos`, `GET /servicos/cliente/{idCliente}`).
- **Atualização de status:** alteração do status do serviço entre `PENDENTE`, `AGENDADO` e `CONCLUIDO` (`PATCH /servicos/{id}/status`, `PATCH /servicos/{id}/concluido`).
- **Histórico do chat:** criação de conversas e registro de mensagens com texto e imagem (`/conversas/...`, `/mensagens`).
- **Controle de acesso:** diferenciação entre clientes e administradores pelo campo `role`.

#### 1.2 Stack tecnológica

| Camada ou finalidade | Tecnologia observada ou documentada |
|---|---|
| Linguagem | Java 17 |
| Framework | Spring Boot 3.5.6 (Spring Web) |
| Persistência | Spring Data JPA / Hibernate |
| Banco de dados | PostgreSQL (driver 42.6.0) |
| Segurança | Spring Security (BCrypt; todas as rotas liberadas com `permitAll`) |
| Build e dependências | Maven (Maven Wrapper `mvnw`) |
| Testes | `spring-boot-starter-test` (JUnit 5, Mockito, AssertJ), sem testes de unidade |
| Cliente da API | Aplicativo React Native/Expo (repositório do frontend) |

O escopo deste trabalho concentra-se no **backend**. O aplicativo mobile será mantido sem alterações e usado para demonstrar que as mesmas funcionalidades continuam operando contra a versão original e a versão refatorada da API.

#### 1.3 Saída do cloc com contagem de linhas

Linhas de código Java em `src/main/java` na versão original, excluindo linhas em branco e comentários. Testes, `target/` e lockfiles ficaram fora da medição.

| Pacote | Arquivos | Linhas de código |
|---|---|---|
| `services` | 4 | 330 |
| `entities` (inclui `SecurityConfig` e `enums`) | 6 | 241 |
| `controllers` | 4 | 197 |
| `dtos` | 11 | 160 |
| `mappers` | 4 | 125 |
| `repositories` | 4 | 35 |
| raiz (`BackendPiscinaApplication`, `CorsConfig`) | 2 | 30 |
| **Total** | **35** | **1.118** |

### 2. Baseline — estado anterior à refatoração

| Indicador | Situação no material recebido | Ação necessária |
|---|---|---|
| Cobertura de testes inicial | Existe apenas `BackendPiscinaApplicationTests.contextLoads()` (`@SpringBootTest`), que depende de um PostgreSQL ativo; não há testes de unidade, logo a cobertura de unidade é 0% | Adicionar JaCoCo, executar `./mvnw test` e registrar o percentual efetivo |
| Avisos e erros de linter | Não mensurados; o `pom.xml` não configura Checkstyle, PMD ou SpotBugs | Configurar Checkstyle/PMD e guardar a saída da primeira execução sobre a versão original |
| Complexidade | Não mensurada | Usar as regras de complexidade ciclomática do PMD e registrar os resultados |
| Comportamento das funcionalidades | Endpoints identificados por inspeção do código e das chamadas feitas pelo app | Testar login, cadastro, edição de conta, criação de serviço e alteração de status antes das mudanças (app + Postman) |
| Dependências | Listadas no `pom.xml`; há um `package-lock.json` sem uso na raiz e o Spring Security é usado apenas para BCrypt | Revisar necessidade das dependências e remover arquivos sem uso |

### 3. Catálogo inicial de smells

| ID | Code smell | Categoria | Localização original | Evidência e impacto |
|---|---|---|---|---|
| CS-01 | **Large Class / serviço com múltiplas responsabilidades** | Inchaços (Bloaters) | `services/ServicoService.java:20-145` | O serviço depende de 5 componentes (`Servico`, `Cliente`, `Conversa` e `Mensagem` repositories + mapper) e concentra criação, validação de posse da conversa, regras de status e geração do resumo do chat; mudanças em qualquer regra afetam a mesma classe. |
| CS-02 | **Primitive Obsession** | Inchaços (Bloaters) | `entities/Cliente.java:18`; `services/ClienteService.java:48`; `dtos/ClienteDTO.java:11` | O papel do usuário é uma `String` livre (`"USER"`), sem validação; qualquer texto vira um papel válido e as regras de acesso dependem de comparação de strings. |
| CS-03 | **Alternative Classes with Different Interfaces** | Abusos de orientação a objetos (Object-Orientation Abusers) | `services/MensagemService.java:32-48` e `services/ConversaService.java:78-106` | Duas classes criam e listam mensagens com assinaturas e regras diferentes (uma valida o dono da conversa e ignora a imagem; a outra aceita imagem e não valida o dono); o comportamento depende de qual endpoint é chamado. |
| CS-04 | **Shotgun Surgery** | Impedidores de mudança (Change Preventers) | `services/ConversaService.java:49-54,68-73`; `mappers/ServicoMapper.java:16-21`; `mappers/ConversaMapper.java:15-21` | A conversão `Cliente → ClienteOutputDTO` é reescrita em quatro lugares, embora exista `ClienteMapper.toOutputDTO` (`mappers/ClienteMapper.java:10`); um novo campo no DTO exige editar vários arquivos. |
| CS-05 | **Duplicate Code** | Dispensáveis (Dispensables) | `services/ConversaService.java:94-105,119-129`; `services/ConversaService.java:82-84,112-114`; `services/ServicoService.java:79-81` | A conversão de imagem para Base64 e a montagem de `MensagemDTO` duplicam `MensagemMapper.toDTO`; a verificação "conversa pertence ao cliente" aparece três vezes. Correções precisam ser replicadas. |
| CS-06 | **Dead Code** | Dispensáveis (Dispensables) | `services/ServicoService.java:123-129`; `services/ClienteService.java:41-52,76-83`; `entities/SecurityConfig.java:14-19`; `services/ConversaService.java:24` | Métodos e dependências sem uso: `atualizarStatus` (substituído por `atualizarStatusDTO`), `createCliente` (o controller salva direto no repositório), uma classe `SecurityConfig` aninhada dentro do serviço, `clienteService` injetado e nunca usado e `conversaMapper` injetado e nunca usado, além de imports sem uso. Aumenta o ruído e engana o leitor. |
| CS-07 | **Comments** | Dispensáveis (Dispensables) | `services/ServicoService.java:49,57,72,96,103-104,112,123`; `services/ConversaService.java:25,31,36`; `dtos/ServicoDTO.java:10,19,30,40` | Comentários de histórico e de conversa (`// ✅ ADICIONAR`, `// ✅ MUDOU`, `// se quiser... me fala`) que descrevem alterações em vez de intenção; ficam desatualizados e poluem o código. |
| CS-08 | **Inappropriate Intimacy / controller acoplado à persistência** | Acopladores (Couplers) | `controllers/ClienteController.java:26,49,70-79` | O controller acessa `ClienteRepository` diretamente, recebe a entidade `Cliente` no corpo da requisição e a devolve na resposta (incluindo a senha), ignorando a camada de serviço e o mapper. |
| CS-09 | **Message Chains** | Acopladores (Couplers) | `services/ConversaService.java:82,112`; `services/ServicoService.java:79`; `mappers/ConversaMapper.java:16-19` | `conversa.getCliente().getIdCliente()` expõe a estrutura interna da entidade; os serviços precisam conhecer a navegação entre objetos para validar a posse da conversa. |

### 4. Plano de Refatoração

#### 4.1 CS-01 — Large Class
**Categoria:** Inchaços (Bloaters)  
**Localização:** services/ServicoService.java

O serviço de solicitações concentra a criação do serviço, a validação de que a conversa pertence ao cliente, as regras de transição de status e a montagem do resumo textual do chat, além de depender de cinco componentes diferentes.

Essa concentração dificulta a compreensão, obriga a criar muitos mocks para testar qualquer método e faz com que alterações em regras independentes aconteçam no mesmo arquivo.

**Refatoração proposta:** aplicar Extract Method no método `create` e Extract Class / Move Method para levar `gerarResumoSimples` a um componente próprio (ex.: `ResumoChatGenerator`), substituindo os números mágicos `30` e `2500` por constantes (Replace Magic Number with Symbolic Constant).

#### 4.2 CS-02 — Primitive Obsession
**Categoria:** Inchaços (Bloaters)  
**Localização:** entities/Cliente.java, services/ClienteService.java e dtos/ClienteDTO.java.

O papel do cliente é representado por uma `String` com valor padrão `"USER"`. Não há restrição sobre os valores aceitos, e o mesmo padrão é repetido no serviço.

Isso permite estados inválidos e espalha o conhecimento sobre os papéis possíveis pelo código.

**Refatoração proposta:** aplicar Replace Type Code with Class (enum `Role`), mantendo o mesmo valor textual persistido e retornado no JSON para não quebrar o aplicativo.

#### 4.3 CS-03 — Alternative Classes with Different Interfaces
**Categoria:** Abusos de orientação a objetos (Object-Orientation Abusers)  
**Localização:** services/MensagemService.java e services/ConversaService.java.

As duas classes oferecem operações equivalentes de criação e listagem de mensagens, com nomes, parâmetros e validações diferentes. `MensagemService.createMensagem` recebe quatro parâmetros soltos, enquanto `ConversaService.salvarMensagem` recebe um DTO.

A duplicação de responsabilidade gera comportamentos inconsistentes e torna incerto qual caminho deve ser mantido.

**Refatoração proposta:** aplicar Move Method para concentrar as operações de mensagem em `MensagemService`, Rename Method para unificar a nomenclatura e Introduce Parameter Object para substituir a lista de parâmetros soltos, mantendo os endpoints existentes como delegações.

#### 4.4 CS-04 — Shotgun Surgery
**Categoria:** Impedidores de mudança (Change Preventers)  
**Localização:** ConversaService.java, ServicoMapper.java e ConversaMapper.java.

A conversão de `Cliente` para `ClienteOutputDTO` é feita manualmente em diversos pontos, apesar de já existir um método para isso em `ClienteMapper`.

Uma alteração nos dados expostos do cliente exigiria modificações em vários arquivos, com risco de inconsistência entre as respostas da API.

**Refatoração proposta:** aplicar Move Method e Inline Method, fazendo todos os pontos delegarem para `ClienteMapper.toOutputDTO` e para `ConversaMapper.toDTO`.

#### 4.5 CS-05 — Duplicate Code
**Categoria:** Dispensáveis (Dispensables)  
**Localização:** services/ConversaService.java e services/ServicoService.java.

A conversão de mensagens para DTO (incluindo a imagem em Base64) e a verificação de posse da conversa são repetidas em diferentes métodos e classes.

A repetição aumenta o esforço de manutenção e o risco de corrigir um ponto e esquecer outro.

**Refatoração proposta:** aplicar Extract Method e reutilizar `MensagemMapper.toDTO`; aplicar Consolidate Duplicate Conditional Fragments para a verificação de posse, centralizada em um único método.

#### 4.6 CS-06 e CS-07 — Dead Code e Comments
**Categoria:** Dispensáveis (Dispensables)  
**Localização:** ServicoService.java, ClienteService.java, ConversaService.java, SecurityConfig.java e ServicoDTO.java.

O código possui métodos, classes, dependências e imports sem uso, além de comentários que registram o histórico de alterações em vez de explicar a intenção do código.

Esses elementos dificultam a leitura e podem induzir a erro sobre o que realmente está em uso.

**Refatoração proposta:** remover código morto (Remove Dead Code / Remove Parameter) e substituir comentários por nomes expressivos (Extract Method / Rename Method), mantendo apenas comentários que expliquem decisões.

#### 4.7 CS-08 — Inappropriate Intimacy
**Categoria:** Acopladores (Couplers)  
**Localização:** controllers/ClienteController.java.

O controller de clientes acessa diretamente o repositório para login e cadastro, recebe e devolve a entidade JPA e contém regras de validação.

Esse comportamento acopla a camada web à persistência, expõe campos internos (como a senha) e impede testar a regra de negócio sem subir a aplicação.

**Refatoração proposta:** aplicar Move Method para levar login e cadastro ao `ClienteService` e Replace Constructor/Entity with DTO na entrada e na saída do endpoint, preservando o formato de JSON consumido pelo aplicativo.

#### 4.8 CS-09 — Message Chains
**Categoria:** Acopladores (Couplers)  
**Localização:** ConversaService.java, ServicoService.java e ConversaMapper.java.

Os serviços navegam por `conversa.getCliente().getIdCliente()` para validar se uma conversa pertence ao cliente.

Essa navegação expõe a estrutura interna das entidades e precisa ser alterada em vários pontos se o relacionamento mudar.

**Refatoração proposta:** aplicar Hide Delegate, criando um método na entidade (ex.: `conversa.pertenceAo(idCliente)`).

### 5. Etapas de Refatoração

#### 5.1 Primeira etapa — Rede de segurança
Antes de qualquer alteração, serão criados testes de caracterização com JUnit 5 e Mockito para `ClienteService`, `ServicoService`, `ConversaService` e `MensagemService`, registrando o comportamento atual, inclusive os defeitos listados na seção 8.4.

#### 5.2 Segunda etapa — Organização das funções
O método `create` de `ServicoService` e a geração do resumo do chat serão divididos com Extract Method, e o código morto e os comentários obsoletos serão removidos.

#### 5.3 Terceira etapa — Movimentação de recursos
As conversões entre entidades e DTOs serão concentradas nos mappers, as operações de mensagem serão unificadas em um único serviço e o acesso ao repositório será retirado do controller.

#### 5.4 Quarta etapa — Organização de dados
O papel do cliente passará a ser um enum, e os números mágicos e as mensagens de erro repetidas serão substituídos por constantes.

#### 5.5 Quinta etapa — Simplificação de condicionais
As verificações repetidas de posse da conversa e as regras de status serão consolidadas, utilizando Guard Clauses quando apropriado.

#### 5.6 Sexta etapa — Simplificação de chamadas
Os métodos com listas longas de parâmetros e nomes fora do padrão Java (`getConteudo_texto`, `getId_mensagem`) serão revisados com Introduce Parameter Object e Rename Method, sem alterar os nomes dos campos no JSON.

### 6. Pull Requests previstos

O desenvolvimento das refatorações será organizado em Pull Requests, seguindo a convenção de nomenclatura Conventional Commits, para facilitar a identificação das alterações e o acompanhamento da evolução do projeto.

| PR | Título | Descrição | Smells |
|---|---|---|---|
| PR 01 | `chore: configurar ferramentas de qualidade` | Configuração de Checkstyle, Spotless, JaCoCo, Husky, commitlint e template de PR. | — |
| PR 02 | `ci: configurar verificações automáticas` | GitHub Actions executando `./mvnw verify` (lint + testes + cobertura) e commitlint. | — |
| PR 03 | `test: adicionar testes de caracterização dos serviços` | Testes de unidade com Mockito protegendo o comportamento atual antes das refatorações. | — |
| PR 04 | `refactor: remover código morto e comentários obsoletos` | Remoção de métodos, classes, dependências, imports e comentários sem uso. | CS-06, CS-07 |
| PR 05 | `refactor: centralizar conversões nos mappers` | Uso de `ClienteMapper`, `ConversaMapper` e `MensagemMapper` em todos os pontos de conversão. | CS-04, CS-05 |
| PR 06 | `refactor: unificar serviços de mensagens` | Concentração das operações de mensagem em um único serviço com interface padronizada. | CS-03 |
| PR 07 | `refactor: reorganizar responsabilidades do serviço de solicitações` | Extração do gerador de resumo, das regras de status e da validação de posse da conversa. | CS-01, CS-09 |
| PR 08 | `refactor: desacoplar controller de clientes da persistência` | Login e cadastro movidos para o serviço, uso de DTOs na entrada e na saída e enum `Role`. | CS-02, CS-08 |
| PR 09 | `docs: atualizar documentação da refatoração` | Atualização do README, CHANGELOG e documentação em `docs/`. | — |

Cada Pull Request deverá apresentar uma descrição das modificações realizadas, os code smells corrigidos, as técnicas de refatoração aplicadas e as evidências de validação das funcionalidades.

### 7. Ferramentas utilizadas

Para auxiliar na análise, refatoração e validação do código, serão utilizadas as seguintes ferramentas:

| Ferramenta | Finalidade |
|---|---|
| Checkstyle (Maven plugin) | Verificar padrões de código e convenções de nomenclatura. |
| PMD (Maven plugin) | Identificar código morto, duplicação e medir complexidade ciclomática. |
| Spotless (google-java-format) | Padronizar a formatação do código. |
| JUnit 5 | Implementar e executar testes unitários. |
| Mockito | Simular repositórios e mappers nos testes dos serviços. |
| JaCoCo | Medir a cobertura de testes (linhas e branches). |
| GitHub Actions | Automatizar lint, testes, cobertura e validação de commits no repositório. |
| Husky + commitlint | Validar mensagens de commit (Conventional Commits) no hook `commit-msg`. |
| Postman | Validar manualmente os endpoints antes e depois das refatorações. |
| ChatGPT e Claude | Auxiliar na identificação de problemas e sugestões de refatoração; todo código gerado será revisado por um integrante. |

## 8. Riscos, limitações e escopo da refatoração

### 8.1 Riscos

Durante o processo de refatoração, alguns riscos deverão ser considerados para evitar problemas no funcionamento da API.

| Risco | Descrição | Medida preventiva |
|---|---|---|
| Alterações no comportamento do sistema | As refatorações podem causar falhas em funcionalidades que já estavam funcionando. | Criar testes de caracterização antes das mudanças e executá-los a cada PR. |
| Contrato com o aplicativo | Renomear campos, rotas ou códigos HTTP pode quebrar o app mobile. | Preservar endpoints, nomes de campos do JSON e códigos de resposta; validar com o app. |
| Autenticação | Mover o login e o cadastro para o serviço pode alterar a forma como a senha é armazenada e comparada. | Manter o comportamento atual e documentar qualquer correção como mudança justificada. |
| Persistência | Alterações em entidades podem gerar mudanças no esquema, já que `ddl-auto=update`. | Não alterar nomes de colunas e tabelas; manter o mapeamento JPA. |
| Dependência do banco nos testes | O teste existente exige PostgreSQL ativo. | Usar testes de unidade com Mockito e um perfil de teste isolado para o contexto Spring. |

### 8.2 Limitações

O trabalho terá como foco a melhoria da qualidade interna do código do backend do Pisciclean, considerando o tempo disponível e os requisitos da disciplina.

As principais limitações são:

- **Cobertura parcial de testes:** a meta será atingir aproximadamente 50% de cobertura, priorizando a camada de serviços.
- **Dependência de serviços externos:** o banco PostgreSQL será simulado com mocks nos testes unitários.
- **Funcionalidades não consumidas pelo app:** os endpoints de conversas e mensagens não são chamados pela versão atual do aplicativo (o chat chama a OpenAI diretamente), por isso serão validados via testes e Postman.

### 8.3 O que não será refatorado

Para manter o foco nos objetivos da disciplina, algumas partes do sistema permanecerão fora do escopo da refatoração.

- **Frontend:** o aplicativo React Native não será alterado; ele será utilizado apenas na demonstração.
- **Banco de dados:** não serão modificadas as tabelas, colunas ou relacionamentos do PostgreSQL.
- **Contrato da API:** rotas, nomes de campos do JSON e códigos de resposta serão preservados.
- **Funcionalidades existentes:** não serão adicionadas novas funcionalidades nem alteradas as regras de negócio.
- **Configuração de segurança:** a política atual (`permitAll`) não será reestruturada.

#### 8.4 Defeitos observados durante o diagnóstico

Os itens abaixo não são code smells, mas foram encontrados durante a análise. Como refatorar não deve alterar o comportamento, só serão corrigidos se a equipe decidir fazê-lo em PR próprio, documentado e justificado no CHANGELOG.

| Defeito | Localização | Efeito |
|---|---|---|
| Senha gravada em texto puro no cadastro, mas criptografada (BCrypt) na edição | `controllers/ClienteController.java:49,77`; `services/ClienteService.java:59` | Se o cliente alterar a senha, o login passa a falhar, pois o login compara texto puro. |
| `deleteCliente` não remove o cliente | `services/ClienteService.java:70-75` | O endpoint responde "Cliente apagado com sucesso." sem apagar nada. |
| Cadastro devolve a entidade completa | `controllers/ClienteController.java:78` | A resposta inclui a senha do cliente. |

O objetivo será preservar as funcionalidades existentes, concentrando as modificações na organização, legibilidade, modularização e manutenibilidade do código.
