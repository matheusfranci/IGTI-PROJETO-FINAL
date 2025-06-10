````markdown
# Deploy Zabbix 7 com PostgreSQL no Kubernetes (GCP GKE Autopilot)

Este repositório contém os comandos e definições para subir um cluster Zabbix 7 com banco PostgreSQL no Kubernetes (GKE Autopilot), utilizando pods simples (não deployments), persistência, e services com IP externo para o frontend.

---

## Pré-requisitos

- Cluster Kubernetes na GCP (GKE Autopilot).
- Namespace `monitoring` criado.
- `kubectl` configurado para o cluster.

---

## 1. Criar namespace monitoring

```bash
kubectl create namespace monitoring
````

---

## 2. Criar PersistentVolumeClaim para PostgreSQL

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: zabbix-postgres-pvc
  namespace: monitoring
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF
```

---

## 3. Criar Pod PostgreSQL com PVC

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: zabbix-postgres
  namespace: monitoring
  labels:
    app: zabbix-postgres
spec:
  containers:
  - name: postgres
    image: postgres:15
    env:
    - name: POSTGRES_DB
      value: zabbix
    - name: POSTGRES_USER
      value: zabbix
    - name: POSTGRES_PASSWORD
      value: zabbixpass
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data/pgdata
    ports:
    - containerPort: 5432
    resources:
      requests:
        cpu: "250m"
        memory: "512Mi"
      limits:
        cpu: "500m"
        memory: "1Gi"
  volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: zabbix-postgres-pvc
EOF
```

---

## 4. Criar Pod Zabbix Server

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: zabbix-server
  namespace: monitoring
  labels:
    app: zabbix-server
spec:
  containers:
  - name: zabbix-server
    image: zabbix/zabbix-server-pgsql:latest
    env:
    - name: DB_SERVER_HOST
      value: zabbix-postgres
    - name: POSTGRES_USER
      value: zabbix
    - name: POSTGRES_PASSWORD
      value: zabbixpass
    ports:
    - containerPort: 10051
    resources:
      requests:
        cpu: "250m"
        memory: "512Mi"
      limits:
        cpu: "500m"
        memory: "1Gi"
EOF
```

---

## 5. Criar Pod Zabbix Web (com toleration para GKE Autopilot taint)

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: zabbix-web
  namespace: monitoring
  labels:
    app: zabbix-web
spec:
  tolerations:
  - key: "cloud.google.com/gke-quick-remove"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: zabbix-web
    image: zabbix/zabbix-web-nginx-pgsql:latest
    env:
    - name: DB_SERVER_HOST
      value: zabbix-postgres
    - name: POSTGRES_USER
      value: zabbix
    - name: POSTGRES_PASSWORD
      value: zabbixpass
    ports:
    - containerPort: 8080
    resources:
      requests:
        cpu: "100m"
        memory: "256Mi"
      limits:
        cpu: "250m"
        memory: "512Mi"
EOF
```

---

## 6. Criar Service LoadBalancer para Zabbix Web (IP externo)

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: zabbix-web-lb
  namespace: monitoring
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: zabbix-web
EOF
```

---

## 7. Verificar o IP externo do serviço

```bash
kubectl get svc zabbix-web-lb -n monitoring
```

Copie o IP em `EXTERNAL-IP` e acesse no navegador:

```
http://<EXTERNAL-IP>/
```

Login padrão:

* Usuário: `Admin`
* Senha: `zabbix`

---

## 8. Monitorar pods

```bash
kubectl get pods -n monitoring
```

---

## Observações

* O banco PostgreSQL utiliza PVC para garantir persistência.
* Os pods possuem requests e limits para facilitar agendamento no GKE Autopilot.
* O pod `zabbix-web` tem toleration para o taint do Autopilot que impede agendamento sem ela.
* Este procedimento evita o uso de Deployment, usando pods simples, conforme solicitado.
* Ajuste os valores de CPU e memória conforme necessário.

---

## Backup dos YAMLs

Para salvar os YAMLs atuais:

```bash
kubectl get pod,zsvc -n monitoring -o yaml > monitoring-resources.yaml
```
