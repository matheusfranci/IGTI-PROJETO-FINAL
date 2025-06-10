Com certeza, irmão! Aqui está um arquivo `.md` prontinho pra você colocar no GitHub. Ele une **toda a instalação do MySQL e GLPI no Kubernetes com persistência de dados**, usando só **kubectl na linha de comando**, sem YAMLs separados.

---

### 📄 `glpi-kubernetes-setup.md`

````markdown
# 🚀 GLPI + MySQL no Kubernetes com Persistência (sem YAML)

Este guia mostra como implantar o **GLPI** com **MySQL 8** no Kubernetes usando apenas `kubectl` pela linha de comando — com armazenamento persistente via PVCs.

---

## 📦 Pré-requisitos

- Cluster Kubernetes configurado (`kubectl` funcionando)
- Permissão para criar `pods`, `services` e `persistentvolumeclaims`

---

## 1️⃣ Criar PVC para MySQL

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF
````

---

## 2️⃣ Criar o Pod MySQL com persistência

### Criar o pod base

```bash
kubectl run glpi-mysql \
  --image=mysql:8 \
  --env="MYSQL_ROOT_PASSWORD=glpipassword" \
  --env="MYSQL_DATABASE=glpidb" \
  --port=3306 \
  --restart=Never
```

### Adicionar o volume persistente

```bash
kubectl patch pod glpi-mysql --type='merge' -p '{
  "spec": {
    "volumes": [{
      "name": "mysql-storage",
      "persistentVolumeClaim": {
        "claimName": "mysql-pvc"
      }
    }],
    "containers": [{
      "name": "glpi-mysql",
      "volumeMounts": [{
        "mountPath": "/var/lib/mysql",
        "name": "mysql-storage"
      }]
    }]
  }
}'
```

### Expor o MySQL internamente

```bash
kubectl expose pod glpi-mysql --port=3306 --name=glpi-mysql
```

---

## 3️⃣ Criar PVC para GLPI

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: glpi-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF
```

---

## 4️⃣ Criar o Pod GLPI com persistência

### Criar o pod base

```bash
kubectl run glpi \
  --image=diouxx/glpi \
  --port=80 \
  --restart=Never
```

### Adicionar o volume persistente

```bash
kubectl patch pod glpi --type='merge' -p '{
  "spec": {
    "volumes": [{
      "name": "glpi-storage",
      "persistentVolumeClaim": {
        "claimName": "glpi-pvc"
      }
    }],
    "containers": [{
      "name": "glpi",
      "volumeMounts": [{
        "mountPath": "/var/www/html",
        "name": "glpi-storage"
      }]
    }]
  }
}'
```

### Expor o GLPI com IP externo

```bash
kubectl expose pod glpi --type=LoadBalancer --port=80 --name=glpi-service
```

---

## ✅ Acesso e Teste

* **GLPI Web:** IP do serviço `glpi-service` (use `kubectl get svc glpi-service`)
* **MySQL:** Internamente via `glpi-mysql:3306`
* **Usuário inicial GLPI:** `glpi / glpi` (senha padrão)

---

## 🔁 Para recriar

Caso precise refazer os pods:

```bash
kubectl delete pod glpi glpi-mysql
kubectl delete svc glpi-service glpi-mysql
```

Os dados serão preservados graças aos PVCs!

---

## ✨ Dica Extra

Para produção, o ideal é usar `Deployments` em vez de `Pods diretos`, e aplicar políticas de backup para os volumes.

---

```

---

Se quiser, posso também converter isso direto num arquivo `.md` para download. Quer?
```
