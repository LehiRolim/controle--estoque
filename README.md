# Sistema de Controle de Estoque com RabbitMQ

## 1. Sobre o projeto

Sistema de gerenciamento de estoque desenvolvido com Node.js, Express, Supabase (PostgreSQL) e RabbitMQ.

A aplicação permite consultar produtos, registrar entradas e saídas de mercadorias e executar processos secundários de maneira assíncrona, utilizando uma arquitetura orientada a eventos.

O RabbitMQ distribui os eventos de movimentação de estoque para três consumidores independentes: Auditoria, Compras e Vitrine.

## 2. Tecnologias utilizadas

- **Node.js:** execução do backend e dos Workers.
- **Express:** desenvolvimento da API REST.
- **Supabase (PostgreSQL):** armazenamento dos dados.
- **RabbitMQ:** comunicação assíncrona entre processos.
- **Docker Compose:** execução local do RabbitMQ.
- **amqplib:** integração do Node.js com RabbitMQ.

## 3. Funcionalidades

### Gerenciamento de produtos

A API permite consultar produtos cadastrados e realizar movimentações de entrada e saída de estoque.

### Worker de Auditoria

Consome mensagens da fila `fila.auditoria` e registra as movimentações na tabela `movimentacoes_estoque`.

Utiliza o identificador único do evento para evitar registros duplicados.

### Worker de Compras

Consome mensagens da fila `fila.compras`.

Quando o estoque atual é menor ou igual ao estoque mínimo, registra um alerta na tabela `alertas_reposicao` e exibe uma notificação simulada no terminal.

### Worker de Vitrine

Consome mensagens da fila `fila.vitrine`.

- Quando o estoque chega a zero, define `disponivel = false`.
- Quando ocorre uma entrada que deixa o estoque positivo, define `disponivel = true`.

## 4. Arquitetura

O sistema utiliza uma Exchange chamada `estoque.eventos`, configurada como `fanout`.

Cada movimentação gera um evento JSON, encaminhado para três filas duráveis:

- `fila.auditoria`
- `fila.compras`
- `fila.vitrine`

Cada Worker processa sua fila independentemente e utiliza confirmação manual (`ack`) após o processamento bem-sucedido.

## 5. Configuração do ambiente

### Pré-requisitos

- Node.js e npm
- Docker Desktop com Docker Compose
- Projeto configurado no Supabase

### Instalar dependências

Na pasta principal do projeto:

```bash
npm install
```

### Configurar variáveis de ambiente

Crie o arquivo `.env` com base no `.env.example`:

```env
SUPABASE_URL=https://seu-projeto.supabase.co
SUPABASE_KEY=sua-chave-de-servidor
RABBITMQ_URL=amqp://admin:admin123@localhost:5672
PORT=3000
```

A chave do Supabase deve permanecer exclusivamente no backend. Não publique o arquivo `.env`.

### Preparar o banco de dados

Os comandos de criação das tabelas estão disponíveis no arquivo:

`sql/estrutura.sql`

Execute esses comandos no SQL Editor de um projeto Supabase novo, conforme necessário.

### Iniciar o RabbitMQ

```bash
docker compose up -d
```

Acesse o painel de gerenciamento:

http://127.0.0.1:15672

Utilize as credenciais definidas no `docker-compose.yml`.

### Configurar a Exchange e as filas

```bash
node iniciarRabbitMQ.js
```

### Iniciar a API

```bash
npm start
```

A API estará disponível em:

http://localhost:3000

### Iniciar os Workers

Execute cada comando em um terminal separado:

```bash
npm run worker:auditoria
```

```bash
npm run worker:compras
```

```bash
npm run worker:vitrine
```

## 6. Rotas da API

### GET /produtos

Consulta os produtos cadastrados no Supabase.

### POST /produtos/:id/movimentar

Realiza uma entrada ou saída de estoque.

Exemplo de requisição:

```json
{
  "tipo": "SAIDA",
  "quantidade": 2
}
```

Após a movimentação, a aplicação atualiza o saldo e solicita a publicação do evento na Exchange `estoque.eventos`.

## 7. Teste de resiliência

O roteiro de demonstração é:

1. Iniciar o RabbitMQ, a API e os três Workers.
2. Desligar somente o Worker de Compras.
3. Realizar duas vendas que deixem um produto abaixo do estoque mínimo.
4. Verificar se a API respondeu normalmente.
5. Abrir o painel RabbitMQ e verificar as mensagens pendentes na `fila.compras`.
6. Reiniciar o Worker de Compras.
7. Confirmar os alertas registrados na tabela `alertas_reposicao` do Supabase.

Esse teste demonstra a retenção de mensagens enquanto um consumidor está indisponível.

## 8. Persistência e confiabilidade

As filas são configuradas como duráveis e as mensagens são publicadas com a opção `persistent: true`.

Os Workers utilizam confirmação manual após o processamento bem-sucedido.

**Limitação atual:** a atualização do estoque e a publicação no RabbitMQ ainda não formam uma transação única. Uma falha entre essas operações pode causar perda de evento. Para garantir essa situação, será necessário implementar uma estratégia de publicação confiável, como Transactional Outbox.

## 9. Estrutura do banco de dados

- `fornecedores`: informações dos fornecedores.
- `produtos`: cadastro e quantidade disponível de produtos.
- `movimentacoes_estoque`: histórico das entradas e saídas.
- `alertas_reposicao`: alertas de produtos com estoque crítico.

## 10. Objetivo acadêmico

Demonstrar a aplicação de uma arquitetura orientada a eventos, utilizando mensageria para desacop