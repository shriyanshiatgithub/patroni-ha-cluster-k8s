# 📘 README – Patroni HA PostgreSQL + PVC + Automated Backups on Kubernetes

This document explains the **complete end-to-end setup** performed:

* Deploy Patroni HA PostgreSQL cluster
* Configure Persistent Volumes
* Insert test data
* Create automated backups using CronJob
* Store backups in PVC
* Verify backup files
* Validate data inside backups

This is a **step-by-step practical implementation guide** with commands and explanations.

---

# 🧱 1️⃣ Environment Overview

Platform used:

* Kubernetes cluster (Killercoda → later moved to WSL)
* 3-node Patroni cluster
* Storage constraints: ~4GB RAM

Architecture built:

```
Patroni StatefulSet (3 pods)
        │
        ├── PVC (pgdata) → database storage
        ├── Service (patronidemo)
        │
        └── CronJob
              │
              └── PVC (backup-pvc)
                     └── .sql backup files
```

---

# 🗂️ 2️⃣ Deploy Patroni Cluster

Apply the Patroni manifest:

```bash
kubectl apply -f k8s/patroni_k8s.yaml
```

Check pods:

```bash
kubectl get pods
```

Expected:

```
patronidemo-0   Running
patronidemo-1   Running
patronidemo-2   Running
```

Check StatefulSet:

```bash
kubectl get statefulset
```

---

# 💾 3️⃣ Persistent Storage Setup

Each pod uses PVC for PostgreSQL data:

```yaml
volumeClaimTemplates:
  - metadata:
      name: pgdata
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 400Mi
```

Check PV:

```bash
kubectl get pv
```

---

# 🔐 4️⃣ Connect to PostgreSQL

Enter primary pod:

```bash
kubectl exec -it patronidemo-0 -- bash
```

Connect using password from config:

```bash
export PGPASSWORD=zalando
psql -h 127.0.0.1 -U postgres
```

---

# 🗄️ 5️⃣ Create Application Database

Inside psql:

```sql
CREATE DATABASE appdb;
```

Switch to it:

```sql
\c appdb
```

---

# 🧪 6️⃣ Create Table

```sql
CREATE TABLE test_backup (
    id SERIAL PRIMARY KEY,
    name TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

# 📊 7️⃣ Insert 50 Rows of Data

```sql
INSERT INTO test_backup (name)
SELECT 'user_' || generate_series(1,50);
```

Verify:

```sql
SELECT COUNT(*) FROM test_backup;
```

Expected:

```
50
```

Exit:

```sql
\q
```

---

# 📦 8️⃣ Create Backup PVC

backup-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: patroni-backup-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply:

```bash
kubectl apply -f backup-pvc.yaml
```

---

# ⏱️ 9️⃣ Create Automated Backup CronJob

backup-cronjob.yaml

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: patroni-backup
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:16
            command:
            - /bin/bash
            - -c
            - |
              TIMESTAMP=$(date +%F-%H-%M)
              PGPASSWORD=zalando pg_dump -h patronidemo -U postgres appdb > /backup/db-$TIMESTAMP.sql
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: patroni-backup-pvc
```

Apply:

```bash
kubectl apply -f backup-cronjob.yaml
```

---

# 🔎 10️⃣ Confirm CronJob is Running

```bash
kubectl get cronjobs
```

```bash
kubectl get jobs
```

```bash
kubectl get pods
```

You should see new backup jobs every 2 minutes.

---

# 📁 11️⃣ Create Debug Pod to View Backups

backup-check-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backup-check
spec:
  containers:
  - name: backup-check
    image: busybox
    command: ["/bin/sh", "-c", "sleep 36000"]
    volumeMounts:
    - name: backup-storage
      mountPath: /backup
  volumes:
  - name: backup-storage
    persistentVolumeClaim:
      claimName: patroni-backup-pvc
```

Apply:

```bash
kubectl apply -f backup-check-pod.yaml
```

---

# 📂 12️⃣ Verify Backup Files

Enter debug pod:

```bash
kubectl exec -it backup-check -- sh
```

List backups:

```bash
ls /backup
```

Expected:

```
db-2026-02-18-06-40.sql
db-2026-02-18-06-42.sql
db-2026-02-18-06-44.sql
```

---

# 🔍 13️⃣ Confirm Data Exists in Backup

View contents:

```bash
tail -n 50 /backup/db-2026-02-18-06-40.sql
```

Check inserted users:

```bash
grep user_ /backup/db-*.sql | wc -l
```

If 3 backups × 50 rows:

```
150
```

✔ Confirms data is included in backups.

---

# 💡 14️⃣ How This System Works

Every 2 minutes:

1. Kubernetes creates a Job
2. Job launches a temporary pod
3. Pod runs pg_dump
4. Backup stored in PVC
5. Pod exits
6. Process repeats

---

# 🛡️ 15️⃣ Persistence Behavior

Database data:

* Stored in pgdata PVC
* Survives pod restart

Backup files:

* Stored in patroni-backup-pvc
* Survive CronJob pod deletion

Note:
If cluster/storage is deleted → data is lost.

---

# 🧪 16️⃣ How to Test Disaster Recovery (Optional)

Delete table:

```sql
DROP TABLE test_backup;
```

Restore from backup:

```bash
psql -h patronidemo -U postgres appdb < db-2026-02-18-06-40.sql
```

---

# 📈 17️⃣ Skills Demonstrated

This setup proves experience in:

* StatefulSets
* PVC storage
* HA PostgreSQL
* Patroni clustering
* CronJobs
* Automated backups
* Kubernetes debugging

---

# 🏁 Final Result

You now have:

* HA PostgreSQL cluster
* Persistent storage
* Automated scheduled backups
* Verifiable restore points
* Production-grade data protection workflow

This is a strong real-world DevOps implementation.
