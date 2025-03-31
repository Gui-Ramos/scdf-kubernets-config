# Execute SCDF locally with kind

### 5. Configurar o postgres como banco

```sh
helm install my-postgres bitnami/postgresql --set auth.username=scdf,auth.password=scdf,auth.database=scdf
```

To connect to your database from outside the cluster execute the following commands:

```sh
kubectl port-forward --namespace default svc/my-postgres-postgresql 5432:5432 &
PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U scdf -d scdf -p 5432
```


### 📌 1. Criar um Cluster Kubernetes Local
🔹 Usando Kind
```sh
kind create cluster --name scdf
```

```sh
kubectl get nodes
```

### 📌 2. Adicionar o Repositório do SCDF no Helm
O SCDF é instalado via Helm Charts. Primeiro, adicione o repositório:

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 📌 3. Instalar o Spring Cloud Data Flow no Kubernetes
Agora, instale o SCDF no cluster:

```sh
helm install my-spring-cloud-dataflow bitnami/spring-cloud-dataflow --version 34.2.1 \
--set database.driverClassName=org.postgresql.Driver \
--set database.url="jdbc:postgresql://my-postgres.default.svc.cluster.local:5432/scdf" \
--set database.username=scdf \
--set database.password=scdf \
--namespace default
```

📌 Isso instalará:

- Spring Cloud Data Flow Server
- Skipper Server
- RabbitMQ (default) como Message Broker
- MySQL como Banco de Dados
- Verifique se os pods foram iniciados:

```sh
kubectl get pods
```

### 📌 4. Acessar a Interface do SCDF
Para acessar a UI do SCDF, execute:

```sh
kubectl port-forward --namespace default svc/my-spring-cloud-dataflow-server 9090:8080
```
Agora, abra no navegador:
👉 http://localhost:9090/dashboard/index.html#/apps




#

#### 📌 Remover o SCDF do Cluster
Se quiser limpar tudo, basta rodar:

```sh
helm uninstall my-spring-cloud-dataflow -n default
kind delete cluster --name scdf
```




#!/bin/bash

set -e

# Criar um cluster KIND
kind create cluster --name scdf-cluster

# Adicionar o repositório Helm do SCDF
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add scdf https://charts.spring.io/scdf
helm repo update

# Criar namespace para o SCDF
kubectl create namespace scdf

# Implantar o PostgreSQL
helm install postgres bitnami/postgresql \
  --namespace scdf \
  --set global.postgresql.auth.postgresPassword=postgres \
  --set global.postgresql.auth.username=scdf \
  --set global.postgresql.auth.password=scdf \
  --set global.postgresql.auth.database=scdf

# Esperar o PostgreSQL estar pronto
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=postgresql -n scdf --timeout=300s

# Implantar o Spring Cloud Data Flow
helm install scdf bitnami/spring-cloud-dataflow \
  --namespace scdf \
  --set server.database.url=jdbc:postgresql://postgres-postgresql.scdf.svc.cluster.local:5432/scdf \
  --set server.database.username=scdf \
  --set server.database.password=scdf \
  --set skipper.database.url=jdbc:postgresql://postgres-postgresql.scdf.svc.cluster.local:5432/scdf \
  --set skipper.database.username=scdf \
  --set skipper.database.password=scdf

# Port Foward
kubectl port-forward --namespace scdf svc/scdf-spring-cloud-dataflow-server 9090:8080

# Exibir os serviços implantados
kubectl get all -n scdf
