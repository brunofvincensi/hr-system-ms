# hr-system-ms
HR System with Microservices
Microservices-based HR System built with Java & Spring Boot. Implements service discovery, API gateway, asynchronous processing (RabbitMQ), and independent persistence per microservice for a scalable HR back-end solution.

# Arquitetura e Decisões Técnicas

## Forma da comunicação entre os microserviços

As chamadas externas chegam inicialmente ao API Gateway, que repassa a requisição para o microserviço correspondente, registrado no Eureka.

- **Comunicação síncrona (REST)**: utilizada quando um microserviço precisa de uma resposta imediata de outro.
- **Comunicação assíncrona (RabbitMQ)**: utilizada para processamentos que podem ocorrer em segundo plano, enviando eventos para filas que serão consumidas posteriormente pelo mesmo microserviço ou por outro.

## Tecnologias escolhidas

- **Java 17**
- **Spring Boot 3.x**
- **PostgreSQL**
- **RabbitMQ**
- **Redis**

## Estratégia para separação de bases de dados

A estratégia que adotei foi a de que cada microserviço possui sua própria base de dados, garantindo isolamento, independência e autonômica de evolução para cada domínio.

## Pontos positivos e negativos e alternativas consideradas nos serviços

### Registry Service

- Serviço que usa o Eureka que faz com que os serviços possam se encontrar sem precisarem saber do endereço fixo dos outros.

### Gateway Service

- Serviço que centraliza as requisições de origem externa.
- A validação das roles dos usuários foi feita nesse serviço, impedindo que as requisições fossem repassadas para os outros serviços caso a role não tenha autorização, fazendo com que centralize essas validações.
- Implementado um cache com Redis das informações do usuário para não precisar consultar a role do usuário em toda requisição, além de passar algumas dessas informações do usuário no header da requisição.
- Quando usuário é alterado é enviado um event pela api de usuário e consumida por este, para invalidar o cache do Redis
- **Ponto negativo**: único ponto de entrada para todos os microserviços, se o Gateway falhar, todo o sistema fica inacessível. Talvez implementar várias instâncias desse serviço para contornar esse problema
- **Sugestões de melhoria**: Implementar a autenticação com JWT ou Oauth2, hoje tem apenas a autorização (roles).

### Employee Service

- Implementado a requisição de cadastrar Escala de Trabalho separado para não ter que repetir a cada novo colaborador e duplicar varias informações na base.
- Implementado a requisição de admissão de colaborador, na qual possue as informações do colaborador, informações para criação de usuário e o id que referência a Escala de Trabalho pré-cadastrada.
- A criação de usuário é feita de forma síncrona (salva colaborador e chama User Service); se houver erro, a transação é abortada.
- **Ponto negativo1**: pode haver inconsistência se ocorrer uma falha inesperada após criar o usuário, mas antes de salvar o colaborador.
- **Alternativa1**: realizar a criação de usuário de forma assíncrona, com mecanismo de compensação (remoção de colaborador em caso de falha).
- **Ponto negativo2**: os serviços filhos não irão controlar autenticação e autorização
- **Alternativa2**: manter apenas a api gateway publica e os serviços em uma network privada para não acessa-los diretemente.

### Time Entry Service

- Cadastro de ponto por data/hora atual para colaboradores.
- Funcionalidades de edição e consulta.
- Criado índice composto por colaborador e data, devido ao alto volume esperado e ao uso frequente desses campos em filtros.
- **Ponto positivo**: otimização de consultas em tabelas grandes desde o início do projeto.
- **Sugestões futuras**: Se a tabela se tornar extremamente grande, pode ser criada uma tabela auxiliar com menos dados, talvez salvando apenas o colaborador e mês e sendo uma fk para a tabela principal, para facilitar filtros iniciais, ajudando na hora de processar as folhas de pagamento

### User Service

- Requisição de criação de usuário chamada pelo Employee Service.
- Armazena apenas informações relacionadas ao usuário (role, senha).
- Ao atualizar um usuário, publica um evento na fila para permitir invalidação de cache em serviços que armazenam dados de usuário.
- **Ponto positivo**: mantém responsabilidade única e integra com mensageria para consistência de cache.

### Payroll Service

- Inicia cálculo de folha de pagamento de forma assíncrona, publicando evento no RabbitMQ.
- Como é assincrona, a requisição retorna imediatamente, apenas com a entidade de movimentação do processamento da folha salva, retornando o id dela para posterior consulta com o status code 202, pois ainda não está concluído
- O próprio serviço consome a fila e processa a folha.
- Faz requisições para a api de colaborador e marcação para ter os dados de "base de cálculo".
- Armazena histórico de execuções com status, usuário solicitante, data de início e fim, para auditoria e consulta.
- Ao finalizar, publica evento para envio de e-mail.
- **Ponto positivo**: evita bloqueio do cliente e garante rastreabilidade do processo.
- **Ponto negativo**: depende de processamento assíncrono, exigindo mecanismos de monitoramento de filas.
- **Sugestões futuras**: Segregar essa tabela, criando uma tabela mais genérica com o intuito apenas de salvar o histórico movimentação do envio do processamento, e uma só para armazenar a folha de pagamento de fato.

### E-mail Service

- Consome eventos de solicitação de envio de e-mails.
- Responsável por efetuar a entrega das notificações.
- **Ponto positivo**: desacopla a responsabilidade de envio, permitindo que outros serviços apenas disparem eventos.

### Common

- Biblioteca comum, compartilhada entre os serviços
- Compartilha dados das roles, constates das informações que vão no header e exceções personalizadas e controller handler padrão das exception

### Project Diagram Link: https://lucid.app/lucidchart/4f39c075-11e3-4779-96b1-34c879567a57/edit?viewport_loc=-1553%2C429%2C4315%2C2129%2C0_0&invitationId=inv_ec149fa5-b2d8-448e-8f10-0de1caea458b
