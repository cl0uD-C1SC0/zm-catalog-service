# catalog-service

Microsserviço de **Catálogo** da Ze Mechanics: peças (com estoque) e serviços (mão de obra).

## Responsabilidades

* Cadastro, consulta, atualização e remoção de peças e serviços.
* Controle de estoque das peças: baixa e devolução.
* Garantir que o estoque nunca fique negativo: uma baixa maior que o disponível é recusada com `400`.

## Arquitetura

```
app/
├── api/v1/          rotas (FastAPI): peça, serviço, health
├── services/        regras de negócio (peca_service, servico_service)
├── repositories/    acesso ao MySQL (SQLAlchemy)
└── domain/models/   Peca, Servicos
```

### Comunicação

| Quem chama | Como | O quê |
|---|---|---|
| os-service | **REST** (síncrono) | Consultar peça e serviço, `remover_quantidade` (baixa de estoque) e `adicionar_quantidade` (devolução, inclusive como **compensação da Saga**) |

O catalog-service não participa das filas do RabbitMQ: ele só responde a chamadas REST. Ele não conhece as OS. Quem guarda quais peças estão em cada OS é o os-service.

## Banco de dados

**MySQL**, schema `catalog_db`, acessado apenas pelo usuário `catalog_user`.

| Tabela | Conteúdo |
|---|---|
| `pecas` | Nome, preço e quantidade em estoque |
| `servicos` | Nome, descrição e preço da mão de obra |

## Endpoints

Prefixo: `/api/v1` · Swagger: `http://localhost:8001/docs` (compose) · `http://<minikube-ip>:30081/docs` (Kubernetes)

| Método | Rota | Descrição |
|---|---|---|
| `POST` | `/peca/adicionar_peca` | Cadastrar uma nova peça |
| `GET` | `/peca/pecas` | Listar todas as peças |
| `GET` | `/peca/consultar_peca/{peca_id}` | Consultar uma peça |
| `PATCH` | `/peca/atualizar_peca/{peca_id}` | Atualizar nome e preço |
| `PATCH` | `/peca/adicionar_quantidade/{peca_id}` | Adicionar quantidade ao estoque |
| `PATCH` | `/peca/remover_quantidade/{peca_id}` | Remover quantidade do estoque (`400` se insuficiente) |
| `DELETE` | `/peca/remover_peca/{peca_id}` | Remover peça |
| `POST` | `/servico/adicionar_servico` | Cadastrar um novo serviço |
| `GET` | `/servico/servicos` | Listar todos os serviços |
| `GET` | `/servico/consultar_servico/{servico_id}` | Consultar um serviço |
| `PATCH` | `/servico/atualizar_servico/{servico_id}` | Atualizar nome, preço e descrição |
| `DELETE` | `/servico/remover_servico/{servico_id}` | Remover um serviço |
| `GET` | `/health/live` | Liveness probe |
| `GET` | `/health/ready` | Readiness probe (testa o MySQL) |

## Variáveis de ambiente

| Variável | Exemplo | Descrição |
|---|---|---|
| `DATABASE_URL` | `mysql+pymysql://catalog_user:catalog_pass@mysql:3306/catalog_db` | Conexão com o MySQL |

## Como rodar

### Local (docker compose)

Com o restante do ambiente, a partir da pasta onde fica o `docker-compose.yml`: `docker compose up --build -d`.

Só este serviço:

```bash
docker build -t catalog-service .
docker run -p 8001:8000 -e DATABASE_URL=... catalog-service
```

### Kubernetes

Os manifestos deste serviço ficam em [`k8s/`](k8s/), no namespace `zm`:

| Arquivo | Conteúdo |
|---|---|
| `deployment.yaml` | 1 réplica, `initContainer` que espera o MySQL, probes em `/api/v1/health/live` e `/ready`, limites de CPU e memória |
| `service.yaml` | NodePort `30081` |
| `secret.yaml` | `DATABASE_URL` |

A infraestrutura compartilhada (namespace `zm`, MySQL, MongoDB, RabbitMQ e Maildev) fica em [`k8s-infra/`](https://github.com/cl0uD-C1SC0/ZeMechanics/tree/feat/microservices/k8s-infra), no repositório principal, e é aplicada uma vez:

```bash
kubectl apply -f k8s-infra/   # uma vez (repositório principal)
kubectl apply -f k8s/         # este serviço (o pipeline faz isso sozinho)
```

Acesso: `http://$(minikube ip):30081/docs`

## CI/CD

Pipeline próprio em [`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml), independente dos outros serviços. Roda em todo push e PR para a `main`.

| Job | Onde roda | O que faz |
|---|---|---|
| `test` | `ubuntu-latest` (GitHub) | `pytest` com cobertura. **Falha se ficar abaixo de 80%** (`--cov-fail-under=80`) e anexa o `coverage.xml` |
| `build` | `ubuntu-latest` (GitHub) | Build da imagem. Na `main`, publica `jayakuza/zm-catalog-service:<sha>` e `:latest` no Docker Hub |
| `deploy` | runner self-hosted (VM com minikube) | Só na `main`: troca a tag do `deployment.yaml` pelo SHA do commit, `kubectl apply -f k8s/` e espera o rollout |

Cada job depende do anterior (`needs`), então uma imagem só chega ao cluster se os testes passarem.

Modelo híbrido: testes e build rodam nos runners do GitHub; só o deploy roda na VM, a única máquina que enxerga o cluster.

## Testes

```bash
pip install -r requirements.txt
DATABASE_URL="sqlite:///./test.db" pytest --cov=app --cov-report=term
```

### Cobertura

Evidência: o relatório sai no log do job **Testes + cobertura** em [Actions](https://github.com/cl0uD-C1SC0/zm-catalog-service/actions), com o `coverage.xml` anexado como artefato. O pipeline barra qualquer commit abaixo de 80%.

```
TOTAL    328 stmts    39 miss    88%
57 passed
```

| Módulo | Cobertura |
|---|---|
| `repositories/PecaRepository.py` | 100% |
| `repositories/ServicosRepository.py` | 100% |
| `services/servico_service.py` | 100% |
| `services/peca_service.py` | 92% |
