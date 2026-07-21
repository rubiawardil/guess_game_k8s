Obs: Esta é uma cópia do [guess_game](https://github.com/rubiawardil/guess_game) modificada para utilizar Kubernetes em sua configuração.

# Jogo de Adivinhação com Flask

Este é um simples jogo de adivinhação desenvolvido utilizando o framework Flask. O jogador deve adivinhar uma senha criada aleatoriamente, e o sistema fornecerá feedback sobre o número de letras corretas e suas respectivas posições.

## Funcionalidades

- Criação de um novo jogo com uma senha fornecida pelo usuário.
- Adivinhe a senha e receba feedback se as letras estão corretas e/ou em posições corretas.
- As senhas são armazenadas utilizando base64.
- As adivinhações incorretas retornam uma mensagem com dicas.

## Configuração Kubernetes/k3d

Arquivos essenciais da configuração:

- `k8s/postgres/`: Secret, PVC, Deployment e Service do banco de dados Postgres.
- `k8s/backend/`: ConfigMap, Deployment (com HPA) e Service da API Flask.
- `k8s/frontend/`: Deployment e Service do React + NGINX.
- `frontend/default.conf`: configuração do NGINX (proxy reverso para o Service `backend`, que já faz o balanceamento de carga entre as réplicas).

## Executando com Kubernetes (k3d)

Essa é a forma de rodar o projeto completo (backend, frontend e banco de dados) em um cluster Kubernetes local. As imagens de backend e frontend já estão publicadas no Docker Hub, então não é necessário instalar Python, Node ou buildar nada, apenas Docker e k3d.

### Pré-requisitos

- Docker Desktop rodando
- k3d instalado
- kubectl instalado

### Como rodar

```bash
k3d cluster create guess-game --port "3000:80@loadbalancer" --k3s-arg "--disable=traefik@server:0"

kubectl apply -f k8s/postgres/
kubectl apply -f k8s/backend/
kubectl apply -f k8s/frontend/
```

Aguarde todos os Pods ficarem prontos:

```bash
kubectl get pods
```

Acesse: **[http://localhost:3000](http://localhost:3000)**

### O que sobe

- `postgres`: banco de dados Postgres 16. Os dados ficam guardados num `PersistentVolumeClaim` (`postgres-pvc`), então sobrevivem mesmo se o Pod for recriado.
- `backend`: Deployment com 2 réplicas da API Flask, todas conectadas ao mesmo banco. Um `HorizontalPodAutoscaler` escala esse Deployment entre 2 e 5 réplicas conforme o uso de CPU.
- `frontend`: builda o React e serve com NGINX. É o único componente exposto para fora do cluster (Service do tipo `LoadBalancer`, mapeado na porta 3000).
- `metrics-server` **e** `local-path-provisioner`: addons que já vêm habilitados por padrão no k3d. O `metrics-server` fornece as métricas de CPU usadas pelo HPA; o `local-path-provisioner` cria o volume usado pelo PVC do Postgres.

### Como funciona o balanceamento de carga

O NGINX (`frontend/default.conf`) encaminha toda chamada de API (`/create`, `/guess/<id>`) para o Service `backend`. Diferente do docker-compose (onde o NGINX lista `backend1` e `backend2` manualmente), aqui o Kubernetes já faz esse balanceamento sozinho: o Service `backend` distribui as requisições entre todas as réplicas do Deployment, não importa quantas existam no momento (2 hoje, até 5 se o HPA escalar).

### Resiliência

Cada Pod do backend tem um `initContainer` que espera o Postgres responder antes de iniciar o Flask, substituindo o `depends_on: condition: service_healthy` do docker-compose. O Postgres e o backend também têm `readinessProbe`/`livenessProbe`; se um Pod travar ou parar de responder, o Kubernetes reinicia ele automaticamente.

### Persistência dos dados

Os dados do Postgres ficam num `PersistentVolumeClaim` (`postgres-pvc`), usando a storage class `local-path` do k3d. `kubectl delete -f k8s/postgres/` mantém o volume; só apagar o PVC ou o cluster (`k3d cluster delete`) reseta o banco.

### Como atualizar cada componente

- **Backend**: troque a tag da imagem em `k8s/backend/deployment.yaml` (`image: rubiawardil/guess-backend:1.0`).
- **Frontend**: troque a tag da imagem em `k8s/frontend/deployment.yaml` (`image: rubiawardil/guess-frontend:1.0`).
- **Postgres**: troque `image: postgres:16` em `k8s/postgres/deployment.yaml`.

Depois de qualquer uma dessas mudanças, reaplique o manifesto correspondente (ex: `kubectl apply -f k8s/backend/`).

### Parar tudo

```bash
k3d cluster delete guess-game
```

### Decisões de design

- **HPA no backend (2 a 5 réplicas, alvo de 50% de CPU)**: precisa que os Pods tenham `resources.requests.cpu` definido, senão o HPA não consegue calcular a porcentagem de uso.
- **Traefik desabilitado na criação do cluster**: o k3d já vem com o Traefik (ingress controller) ocupando a porta 80 por padrão, o que conflita com o Service `LoadBalancer` do frontend (também na porta 80). Como não usamos ingress-controller, desabilitamos o Traefik em vez de usar uma porta alternativa.
- `initContainer` **no backend**: Kubernetes não tem um equivalente direto ao `depends_on: condition: service_healthy` do docker-compose. Um `initContainer` que espera o Postgres responder antes de iniciar o Flask recria esse comportamento.
- **Imagens multi-arquitetura (amd64 + arm64)**: builds feitos com `docker buildx`, para funcionar tanto em máquinas Intel quanto Apple Silicon sem precisar rebuildar.

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
