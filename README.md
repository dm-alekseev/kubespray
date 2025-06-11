# Структура проекта
```
kustomization/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── index.html
    ├── test/
    │   ├── kustomization.yaml
    │   └── index.html
    └── prod/
        ├── kustomization.yaml
        └── index.html
```
1. Базовые файлы

base/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html-volume
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
      volumes:
        - name: html-volume
          configMap:
            name: index-html-cm
```

base/service.yaml
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer # или NodePort/ClusterIP по необходимости
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

base/kustomization.yaml
```yaml

resources:
  - deployment.yaml
  - service.yaml

configMapGenerator:
  - name: index-html-cm # содержимое переопределяется в окружениях через патчи или генерацию.
```
2. Окружения (overlays)

overlays/dev/kustomization.yaml
```yaml
resources:
  - ../../base

namePrefix: dev-

configMapGenerator:
  - name: index-html-cm
    files:
      - index.html=../../overlays/dev/index.html

overlays/test/kustomization.yaml
resources:
  - ../../base

namePrefix: test-

configMapGenerator:
  - name: index-html-cm
    files:
      - index.html=../../overlays/test/index.html
```

overlays/prod/kustomization.yaml
```yaml
resources:
  - ../../base

namePrefix: prod-

configMapGenerator:
  - name: index-html-cm
    files:
      - index.html=../../overlays/prod/index.html
```

3. HTML файлы с русской кодировкой
overlays/dev/index.html
```html
<html>
<head>
<meta charset="UTF-8">
<title>Dev Environment</title>
</head>
<body>
<h1>Это окружение DEV</h1>
</body>
</html>
overlays/test/index.html
```


Как применить:
Для деплоя в нужное окружение выполните:
```bash
kubectl apply -k overlays/dev/      
kubectl apply -k overlays/test/    
kubectl apply -k overlays/prod/     