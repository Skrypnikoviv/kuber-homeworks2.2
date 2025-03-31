# Домашнее задание к занятию «Хранение в K8s. Часть 2»

## Задание 1: Работа с локальным PersistentVolume

### 1. Создаем манифесты

**1.1. PV (local-pv.yaml):**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - ilko-virtualbox
```

**1.2. PVC (local-pvc.yaml):**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 1Gi
```

**1.3. Deployment (deployment.yaml):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shared-storage-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shared-storage
  template:
    metadata:
      labels:
        app: shared-storage
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c", "while true; do echo $(date) >> /shared-data/log.txt; sleep 5; done"]
        volumeMounts:
        - name: shared-storage
          mountPath: /shared-data
      - name: multitool
        image: wbitt/network-multitool
        volumeMounts:
        - name: shared-storage
          mountPath: /shared-data
      volumes:
      - name: shared-storage
        persistentVolumeClaim:
          claimName: local-pvc
```

### 2. Применяем манифесты
```bash
kubectl apply -f local-pv.yaml
kubectl apply -f local-pvc.yaml
kubectl apply -f deployment.yaml
```

### 3. Проверяем работу
```bash
# Проверяем созданные ресурсы
kubectl get pv,pvc,pods

# Смотрим логи из multitool
kubectl exec -it shared-storage-deployment-b9487755-v88qw -c multitool -- cat /shared-data/log.txt
```
![image](https://github.com/user-attachments/assets/65dc5158-e175-4425-9da5-c8176999e38e)

### 4. Удаляем Deployment и PVC
```bash
kubectl delete deployment shared-storage-deployment
kubectl delete pvc local-pvc
```
![image](https://github.com/user-attachments/assets/b1d7250e-6708-4dd7-a911-2914a6a3e2f4)

### 5. Проверяем состояние PV
```bash
kubectl get pv
# PV должен остаться в статусе Released
```

### 6. Проверяем файл на ноде
```bash
# На ноде проверяем содержимое /mnt/data/log.txt
cat /mnt/data/log.txt
```
![image](https://github.com/user-attachments/assets/2c5f84e5-88a0-4e50-acaa-9db9679f67a1)

### 7. Удаляем PV
```bash
kubectl delete pv local-pv
# Файл останется на ноде, так как PV был с политикой Retain
```
![image](https://github.com/user-attachments/assets/6d14c227-f1b9-4c82-ba78-73fcdf041027)

## Задание 2: Работа с NFS

### 1. Настраиваем NFS на MicroK8S

```bash
# Включаем NFS
microk8s enable nfs
```

### 2. Создаем манифесты

**2.1. StorageClass (nfs-sc.yaml):**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-sc
provisioner: microk8s.io/nfs
```

**2.2. PVC (nfs-pvc.yaml):**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-sc
  resources:
    requests:
      storage: 1Gi
```

**2.3. Deployment (nfs-deployment.yaml):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-app
  template:
    metadata:
      labels:
        app: nfs-app
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        volumeMounts:
        - name: nfs-storage
          mountPath: /nfs-data
      volumes:
      - name: nfs-storage
        persistentVolumeClaim:
          claimName: nfs-pvc
```

### 3. Применяем манифесты
```bash
kubectl apply -f nfs-sc.yaml
kubectl apply -f nfs-pvc.yaml
kubectl apply -f nfs-deployment.yaml
```

### 4. Проверяем работу
```bash
# Записываем файл
kubectl exec -it <pod-name> -- sh -c "echo 'Test NFS' > /nfs-data/test.txt"

# Читаем файл
kubectl exec -it <pod-name> -- cat /nfs-data/test.txt
```

## Выводы

1. В первом задании с локальным PV:
   - После удаления Deployment и PVC, PV перешел в состояние Released, но данные сохранились
   - После удаления PV файл остался на ноде, так как использовалась политика Retain

2. Во втором задании с NFS:
   - Динамическое выделение PV работает корректно
   - Данные успешно записываются и читаются из NFS-тома

Манифесты и команды предоставлены в соответствующих разделах.
