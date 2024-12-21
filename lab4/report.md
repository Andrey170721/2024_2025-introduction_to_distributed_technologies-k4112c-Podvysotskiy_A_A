University: [ITMO University](https://itmo.ru/ru/)
Faculty: [FICT](https://fict.itmo.ru)
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies)
Year: 2023/2024
Group: K4112c
Author: Podvysotskiy Andrey Anatolievich
Lab: Lab4
Date of create: 10.09.2024
Date of finished: 19.09.2024

# Отчет по лабораторной работе №4

## Ход работы

### Шаг 1

Запустим Minikube с Calico и двумя нодами

```bash
minikube start --network-plugin=cni --cni=calico --nodes=2 -p multinode-demo
```

![1](photoReport/1.png)

Проверим что создались две ноды

```bash
kubectl get nodes
```

![2](photoReport/2.png)


Проверbv установку Calico в кластере

```bash
kubectl get pods -l k8s-app=calico-node -A
```

![1-extra](photoReport/1-extra.png)

### Шаг 2

Теперь назначим метки нодам по их географическому расположению

```bash
kubectl label nodes multinode-demo location=us-east
kubectl label nodes multinode-demo-m02 location=us-west
```

![3](photoReport/3.png)

### Шаг 3

Теперь разработаем манифест IPPool для Calico, который будет назначать IP-адреса подам на основе меток нод.

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ippool-zone1
spec:
  cidr: 192.168.0.0/24
  ipipMode: Always
  natOutgoing: true
  nodeSelector: zone == "ru-east"
---
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ippool-zone2
spec:
  cidr: 192.168.1.0/24
  ipipMode: Always
  natOutgoing: true
  nodeSelector: zone == "ru-west"

```

Применим

```bash
kubectl apply -f ippool.yaml
```

![4](photoReport/4.png)

Удалим дефолтный ippool

![3-extra](photoReport/3-extra.png)

Теперь проверим что пулы включились

```bash
calicoctl get ippool -o wide
```

![2-extra](photoReport/2-extra.png)

### Шаг 4

Теперь создадим манифесты deployment и service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: location
                    operator: In
                    values:
                      - us-east
              - matchExpressions:
                  - key: location
                    operator: In
                    values:
                      - us-west
      containers:
        - name: frontend
          image: ifilyaninitmo/itdt-contained-frontend:master
          ports:
            - containerPort: 3000
          env:
            - name: REACT_APP_USERNAME
              value: "Andrey"
            - name: REACT_APP_COMPANY_NAME
              value: "ITMO"
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: NodePort
```

Применим

```bash
kubectl apply -f deployment.yaml
```

![5](photoReport/5.png)

```bash
kubectl apply -f service.yaml
```

![6](photoReport/6.png)

### Шаг 5

Теперь проверим их работу

```bash
kubectl get pods -o wide
```

![4-extra](photoReport/4-extra.png)

## Схема

![diagram](diagram.png)