# Coworking Space Service Extension

The Coworking Space Service is a set of APIs that enables users to request one-time tokens and administrators to authorize access to a coworking space. This service follows a microservice pattern and the APIs are split into distinct services that can be deployed and managed independently of one another.

## Getting Started

### Setup

#### 0. Create an EKS Cluster

Install eksctl(opens in a new tab) and use it to create an EKS cluster.

1. Ensure the AWS CLI is configured correctly.

```bash
aws sts get-caller-identity
```

2. Create an EKS Cluster

```bash
eksctl create cluster --name hoand-cluster --region us-east-1 --nodegroup-name my-nodes --node-type t3.small --nodes 1 --nodes-min 1 --nodes-max 2
aws eks --region us-east-1 update-kubeconfig --name hoand-cluster

```

3. Update the Kubeconfig

```bash
kubectl config current-context
kubectl config view
```

#### 1. Configure a Database

Set up a Postgres database

1. Create a file pvc.yaml on your local machine, with the following content.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

2. Create PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hoand-manual-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gp2
  hostPath:
    path: "/mnt/data"
```

3. Create postgresql.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
spec:
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql
          image: postgres:latest
          env:
            - name: POSTGRES_DB
              value: hoanddb
            - name: POSTGRES_USER
              value: hoand
            - name: POSTGRES_PASSWORD
              value: hoand
          ports:
            - containerPort: 5432
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgresql-storage
      volumes:
        - name: postgresql-storage
          persistentVolumeClaim:
            claimName: postgresql-pvc
```

4.  run commands

```bash
	kubectl apply -f ./database/pvc.yaml
	kubectl apply -f ./database/pv.yaml
	kubectl apply -f ./database/postgresql.yaml
```

3. Test Database Connection

- Connecting Via Port Forwarding

```bash
kubectl port-forward --namespace default svc/<SERVICE_NAME>-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432
```

- Connecting Via a Pod

```bash
kubectl exec -it <POD_NAME> bash
PGPASSWORD="<PASSWORD HERE>" psql postgres://postgres@<SERVICE_NAME>:5432/postgres -c <COMMAND_HERE>
```

4. Run Seed Files
   We will need to run the seed files in `db/` in order to create the tables and populate them with data.

```bash
		kubectl port-forward service/postgresql-service 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < <FILE_NAME.sql>
```

### 2. Running the Analytics Application Locally

In the `analytics/` directory:

1. Install dependencies

```bash
pip install -r requirements.txt
```

2. Run the application (see below regarding environment variables)

```bash
<ENV_VARS> python app.py
```

There are multiple ways to set environment variables in a command. They can be set per session by running `export KEY=VAL` in the command line or they can be prepended into your command.

- `DB_USERNAME`
- `DB_PASSWORD`
- `DB_HOST` (defaults to `127.0.0.1`)
- `DB_PORT` (defaults to `5432`)
- `DB_NAME` (defaults to `postgres`)

If we set the environment variables by prepending them, it would look like the following:

```bash
DB_USER=username_here DB_PASSWORD=password_here python app.py
```

The benefit here is that it's explicitly set. However, note that the `DB_PASSWORD` value is now recorded in the session's history in plaintext. There are several ways to work around this including setting environment variables in a file and sourcing them in a terminal session.

3. Verifying The Application

- Generate report for check-ins grouped by dates
  `curl <BASE_URL>/api/reports/daily_usage`

- Generate report for check-ins grouped by users
  `curl <BASE_URL>/api/reports/user_visits`

## Project Instructions

1. Set up a Postgres database with a Helm Chart
2. Create a `Dockerfile` for the Python application. Use a base image that is Python-based.
3. Write a simple build pipeline with AWS CodeBuild to build and push a Docker image into AWS ECR
4. Create a service and deployment using Kubernetes configuration files to deploy the application
5. Check AWS CloudWatch for application logs

```Dockerfile
FROM python:3.11-slim

WORKDIR /usr/src/app

COPY . .

RUN apt update -y && \
    pip install --no-cache-dir -r requirements.txt

COPY . .
ENV DB_USER=hoand
ENV DB_PASSWORD=hoand
ENV DB_HOST=127.0.0.1
ENV DB_PORT=5432
ENV DB_NAME=hoanddb
EXPOSE 5153

CMD [ "python", "./app.py" ]
```

4.  run commands

```bash
cd ./deployment
kubectl apply -f coworking.yaml
```
