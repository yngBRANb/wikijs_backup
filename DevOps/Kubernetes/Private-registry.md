---
title: Использование приватных container registry в Kubernetes
description: 
published: true
date: 2025-07-21T05:37:28.508Z
tags: devops, ghcr, glcr, kubernetes
editor: markdown
dateCreated: 2025-07-21T05:35:29.189Z
---

# Введение

В любом production кластере требуется использование артефактов из приватных registry, для этого требуется чтобы K8s умел получать к ним доступ. Тут речь пойдет о ghcr, но это так же применимо к glcr.

В ghcr для этого требуется сгенерировать classic PAT (Personal access token) с правами на запись и удаление packages. После чего из этого токена и вашего GitHub username нужно создать секрет.

> Secret должен находится в том же namespace, что и deployment's которые будут его использовать
{.is-warning}


Пример:

```plaintext
kubectl create secret docker-registry ghcr-secret \
  --namespace test
  --docker-server=ghcr.io \
  --docker-username=<your-github-username> \
  --docker-password=<your-personal-access-token> \
  --docker-email=your-email@example.com
```

Пример deployment с использованием артефакта из приватного repo:

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
  namespace: test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: notification
  template:
    metadata:
      labels:
        app: notification
    spec:
      containers:
      - name: notification
        image: ghcr.io/firysksta/crispy-train/notification-service:main-abc1234
        ports:
        - containerPort: 8080
      imagePullSecrets:
      - name: ghcr-auth  # Ссылка на созданный Secret
```