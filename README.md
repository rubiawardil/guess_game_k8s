# Jogo de Adivinhação com Flask

Este é um simples jogo de adivinhação desenvolvido utilizando o framework Flask. O jogador deve adivinhar uma senha criada aleatoriamente, e o sistema fornecerá feedback sobre o número de letras corretas e suas respectivas posições.

## Funcionalidades

- Criação de um novo jogo com uma senha fornecida pelo usuário.
- Adivinhe a senha e receba feedback se as letras estão corretas e/ou em posições corretas.
- As senhas são armazenadas utilizando base64.
- As adivinhações incorretas retornam uma mensagem com dicas.

## Configuração Docker/NGINX

Arquivos essenciais da configuração:

- **`docker-compose.yml`** (raiz): orquestra todos os serviços (Postgres, backends, frontend).
- **`Dockerfile`** (raiz): build do container do backend Flask.
- **`frontend/Dockerfile`**: build do frontend React e da imagem que serve com NGINX.
- **`frontend/default.conf`**: configuração do NGINX (proxy reverso e balanceamento de carga entre os backends).

## Executando com Docker Compose

Essa é a forma de rodar o projeto completo (backend, frontend e banco de dados), não sendo necessário instalar Python, Node ou Postgres na sua máquina, apenas o Docker.

### Pré-requisitos

- Docker instalado (o Docker Desktop já vem com o Compose incluso. Em Linux com Docker Engine "puro", pode ser necessário instalar o pacote `docker-compose-plugin` à parte)

### Como rodar

```bash
docker compose up -d --build
```

Acesse: **http://localhost:3000**

### O que sobe

- **`postgres`**: banco de dados Postgres 16. Os dados ficam guardados num volume (`pgdata`), então sobrevivem mesmo se o container for recriado.
- **`backend1` e `backend2`**: duas cópias idênticas da API Flask, ambas conectadas ao mesmo banco. Existem duas cópias justamente para permitir balanceamento de carga.
- **`frontend`**: builda o React e serve com NGINX. É o único container que expõe uma porta pro seu navegador (a 3000).

### Como funciona o balanceamento de carga

O NGINX (`frontend/default.conf`) lista `backend1` e `backend2` num bloco `upstream`. Toda chamada de API (`/create`, `/guess/<id>`) passa por esse NGINX, que alterna entre os dois backends a cada request. Se um deles cair, o outro continua respondendo.

### Resiliência

Todos os serviços têm `restart: unless-stopped` — se algum travar, o Docker sobe ele de novo automaticamente. O Postgres também tem um `healthcheck`, e os backends só sobem depois que o banco realmente está pronto para aceitar conexões.

### Persistência dos dados

Os dados do Postgres ficam no volume `pgdata`. `docker compose down` mantém esses dados; só `docker compose down -v` apaga o volume e reseta o banco.

### Como atualizar cada componente

- **Postgres**: troque `image: postgres:16` no `docker-compose.yml` pela versão desejada.
- **Backend**: troque `FROM python:3.12-slim` no `Dockerfile` da raiz.
- **Frontend**: troque `FROM node:18-alpine` ou `FROM nginx:alpine` no `frontend/Dockerfile`.

Depois de qualquer uma dessas mudanças, rode de novo:

```bash
docker compose up -d --build
```

### Parar tudo

```bash
docker compose down
```

### Decisões de design

- **Duas cópias fixas do backend (`backend1` e `backend2`)**: em vez de deixar o Docker escalar um serviço sozinho, criamos duas cópias com nomes fixos. Isso deixa simples e confiável pro NGINX saber exatamente pra quais dois endereços mandar os pedidos, sem configuração extra.
- **`REACT_APP_BACKEND_URL` vazio no frontend**: assim, as chamadas da API são feitas pro mesmo endereço que carregou a página (ex: `/create`), em vez de um endereço fixo de backend. Isso evita problemas de CORS e mantém o frontend independente de onde o backend está rodando.
- **`healthcheck` no Postgres**: garante que os backends só iniciem depois que o banco de dados estiver realmente pronto para receber conexões, evitando erros de inicialização.
- **Volume nomeado `pgdata`**: garante que os dados do jogo não sejam perdidos se o container do Postgres for recriado.

## Como Jogar

### 1. Criar um novo jogo

Acesse a url do frontend http://localhost:3000

Digite uma frase secreta

Envie

Salve o game-id

### 2. Adivinhar a senha

Acesse a url do frontend http://localhost:3000

Vá para o endponint breaker

entre com o game_id que foi gerado pelo Creator

Tente adivinhar

## Estrutura do Código

### Rotas:

- **`/create`**: Cria um novo jogo. Armazena a senha codificada em base64 e retorna um `game_id`.
- **`/guess/<game_id>`**: Permite ao usuário adivinhar a senha. Compara a adivinhação com a senha armazenada e retorna o resultado.

### Classes Importantes:

- **`Guess`**: Classe responsável por gerenciar a lógica de comparação entre a senha e a tentativa do jogador.
- **`WrongAttempt`**: Exceção personalizada que é levantada quando a tentativa está incorreta.

## Licença

Este projeto está licenciado sob a [MIT License](LICENSE).
