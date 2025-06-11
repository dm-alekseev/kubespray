```bash
sudo apt-get update -y
sudo apt install software-properties-common
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt-get update -y
sudo apt-get install git pip python3.13 -y

sudo -i
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3.9 get-pip.py



# RETURN TO USER
VENVDIR=kubespray-venv
KUBESPRAYDIR=kubespray
python3 -m venv $VENVDIR
source $VENVDIR/bin/activate
cd $KUBESPRAYDIR
pip install -U -r requirements.txt
 ./contrib/offline/generate_list.shgit clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray/
pip3.13 install -r requirements.txt

# Copy ``inventory/sample`` as ``inventory/mycluster``
cp -rfp inventory/sample inventory/mycluster


# Copy private ssh key to ansible host
ssh-keygen
ssh-copy-id vboxuser@192.168.100.34
ssh-copy-id vboxuser@192.168.100.35
ssh-copy-id vboxuser@192.168.100.37
sudo chmod 600 ~/.ssh/id_rsa

ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml -b -v &


mkdir ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

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