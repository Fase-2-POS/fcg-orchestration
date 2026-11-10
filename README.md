# 🎮 FIAP Cloud Games — Orquestração (Fase 2)

Repositório de orquestração do **Tech Challenge — Fase 2 (FIAP)**.

Na Fase 1 a FCG era uma API monolítica. Nesta fase ela foi refatorada em **quatro microsserviços independentes**, cada um no seu próprio repositório, que se comunicam de forma **assíncrona** pelo **RabbitMQ**.

## 📦 Repositórios

| Repositório | Responsabilidade |
|---|---|
| [fcg-users-api](https://github.com/Fase-2-POS/fcg-users-api) | Cadastro, login (JWT) e autorização de usuários |
| [fcg-catalog-api](https://github.com/Fase-2-POS/fcg-catalog-api) | CRUD de jogos, promoções, início da compra e biblioteca |
| [fcg-payments-api](https://github.com/Fase-2-POS/fcg-payments-api) | Simulação do pagamento |
| [fcg-notifications-api](https://github.com/Fase-2-POS/fcg-notifications-api) | Simulação de e-mails (log no console) |
| [fcg-orchestration](https://github.com/Fase-2-POS/fcg-orchestration) | Docker Compose, manifestos Kubernetes e RabbitMQ (este repositório) |

## 🔄 Fluxos orientados a eventos

**Cadastro de usuário**

```
UsersAPI ──UserCreatedEvent──► RabbitMQ ──► NotificationsAPI (e-mail de boas-vindas)
```

**Compra de jogo**

```
CatalogAPI ──OrderPlacedEvent──► RabbitMQ ──► PaymentsAPI
PaymentsAPI ──PaymentProcessedEvent──► RabbitMQ ──► CatalogAPI (Approved: adiciona à biblioteca)
                                                └─► NotificationsAPI (Approved: e-mail de confirmação)
```

Regra de pagamento simulado: valor **até R$ 200** é aprovado, acima disso é rejeitado.

## 🛠️ Tecnologias

.NET 8, ASP.NET Core, Entity Framework Core + SQLite, JWT, MassTransit, RabbitMQ, Docker, Docker Compose e Kubernetes (Docker Desktop).

## 📋 Pré-requisitos

- Docker Desktop com o **Kubernetes ativado** (Settings → Kubernetes → Enable Kubernetes)
- Git

## 📥 Clonando os repositórios

Os cinco repositórios precisam estar **na mesma pasta**, porque o `docker-compose.yml` usa caminhos como `../fcg-users-api`:

```bash
mkdir fcg-fase2
cd fcg-fase2
git clone https://github.com/Fase-2-POS/fcg-orchestration.git
git clone https://github.com/Fase-2-POS/fcg-users-api.git
git clone https://github.com/Fase-2-POS/fcg-catalog-api.git
git clone https://github.com/Fase-2-POS/fcg-payments-api.git
git clone https://github.com/Fase-2-POS/fcg-notifications-api.git
```

Estrutura final:

```
fcg-fase2/
├── fcg-orchestration/
├── fcg-users-api/
├── fcg-catalog-api/
├── fcg-payments-api/
└── fcg-notifications-api/
```

---

## 🐳 Executando com Docker Compose

```bash
cd fcg-orchestration
docker compose up -d --build
docker compose ps
```

| Serviço | Endereço |
|---|---|
| RabbitMQ (painel) | http://localhost:15672 (usuário `fcg`, senha `fcg123`) |
| Users API | http://localhost:5001/swagger |
| Catalog API | http://localhost:5002/swagger |
| Payments API | http://localhost:5003 |
| Notifications API | http://localhost:5004 |

Para parar:

```bash
docker compose down
```

## 🧪 Testando os fluxos

**1. Fluxo de cadastro**

1. No Users API, execute `POST /api/auth/register`:
   ```json
   { "nome": "Maria Teste", "email": "maria@teste.com", "senha": "Senha@123" }
   ```
2. Veja o e-mail de boas-vindas no log:
   ```bash
   docker compose logs notifications-api
   ```

**2. Cadastrar jogos (administrador)**

1. No Users API, faça login com `admin@fcg.com` / `Admin@123` e copie o `token`.
2. No Catalog API, clique em **Authorize**, cole o token e cadastre dois jogos em `POST /api/jogos`:
   ```json
   { "titulo": "Space Adventure", "descricao": "Aventura espacial", "preco": 59.90 }
   ```
   ```json
   { "titulo": "Mega Racing Ultimate", "descricao": "Corrida", "preco": 299.90 }
   ```

**3. Fluxo de compra**

1. No Users API, faça login com o usuário criado (`maria@teste.com`) e copie o token.
2. No Catalog API, clique em **Authorize** com esse token e execute `POST /api/aquisicoes/1` (jogo de R$ 59,90). A resposta é **202 Accepted**.
3. Veja os logs:
   ```bash
   docker compose logs payments-api
   docker compose logs catalog-api
   docker compose logs notifications-api
   ```
4. Em `GET /api/aquisicoes/minhas` o jogo aparece na biblioteca.
5. Repita com `POST /api/aquisicoes/2` (R$ 299,90): o pagamento é **rejeitado**, o jogo não entra na biblioteca e nenhum e-mail é enviado.

---

## ☸️ Deploy no Kubernetes (Docker Desktop)

**1. Gerar as imagens**

```bash
cd fcg-orchestration
docker compose build
```

Isso cria as imagens `fcg-users-api:latest`, `fcg-catalog-api:latest`, `fcg-payments-api:latest` e `fcg-notifications-api:latest`, usadas nos Deployments.

**2. Aplicar os manifestos**

```bash
cd k8s
kubectl apply -f .
```

**3. Verificar os Pods**

```bash
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get configmaps
kubectl get secrets
```

Todos os Pods devem ficar com status **Running**.

**4. Acessar os serviços**

Os Services são do tipo `ClusterIP` (acesso interno). Para acessar pelo navegador, use `port-forward`, cada um em um terminal:

```bash
kubectl port-forward service/users-api 5001:80
kubectl port-forward service/catalog-api 5002:80
kubectl port-forward service/rabbitmq 15672:15672
```

Depois, repita os testes da seção anterior. Os logs ficam em:

```bash
kubectl logs deployment/payments-api
kubectl logs deployment/catalog-api
kubectl logs deployment/notifications-api
```

**5. Remover tudo do cluster**

```bash
kubectl delete -f .
```

### 📄 Manifestos

| Arquivo | Recurso |
|---|---|
| `*-deployment.yaml` | **Deployment** que gerencia os Pods de cada serviço |
| `*-service.yaml` | **Service** com o nome usado na comunicação interna (`users-api`, `catalog-api`, `payments-api`, `notifications-api`, `rabbitmq`) |
| `*-configmap.yaml` | **ConfigMap** com dados não sensíveis (host do RabbitMQ, nomes das filas, limite de pagamento, issuer do JWT) |
| `*-secret.yaml` | **Secret** com dados sensíveis (connection string, chave JWT, usuário e senha do RabbitMQ) |

Os manifestos de cada microsserviço também estão na pasta `/k8s` do seu próprio repositório. Aqui eles foram reunidos com os do RabbitMQ para permitir o deploy completo com um único `kubectl apply -f .`.

> Os Secrets estão no repositório apenas por ser um projeto acadêmico. Em um ambiente real eles não seriam versionados.

### ⚠️ Pod com `ErrImageNeverPull` ou `ImagePullBackOff`

Significa que o cluster não encontrou a imagem local. No Docker Desktop, acesse **Kubernetes → Edit cluster** e selecione o tipo **kubeadm**, que usa as mesmas imagens do Docker. Depois rode `docker compose build` e `kubectl apply -f .` novamente.

---

Feito por **Milena Torres**
