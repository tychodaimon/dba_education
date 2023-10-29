# Разворачиваем Postgres в minikube

#### Ставим docker
```
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo   "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Ставим minikube
```
wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube
minikube version
minikube version: v1.31.2
commit: fd7ecd9c4599bef9f04c0986c4a0187f98a4396e
```

#### Ставим kubectl
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version -o json --client
{
  "clientVersion": {
    "major": "1",
    "minor": "28",
    "gitVersion": "v1.28.3",
    "gitCommit": "a8a1abc25cad87333840cd7d54be2efaf31a3177",
    "gitTreeState": "clean",
    "buildDate": "2023-10-18T11:42:52Z",
    "goVersion": "go1.20.10",
    "compiler": "gc",
    "platform": "linux/amd64"
  },
  "kustomizeVersion": "v5.0.4-0.20230601165947-6ce0bf390ce3"
}
```

#### Запускаем minikube и переключаем окружение на его докер
```
sudo chmod 0755 /var/run/docker.sock
sudo chown dba9:dba9 /var/run/docker.sock
minikube start --vm-driver=docker
minikube docker-env
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.49.2:2376"
export DOCKER_CERT_PATH="/home/dba9/.minikube/certs"
export MINIKUBE_ACTIVE_DOCKERD="minikube"
docker ps
```

#### Запускаем докер дашборд и делаем его доступным извне
```
minikube dashboard
kubectl proxy --address='0.0.0.0' --disable-filter=true
```

#### Разворачиваем postgres
```
cat postgres.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  type: NodePort
  ports:
   - port: 5432
  selector:
    app: postgres

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-statefulset
spec:
  serviceName: "postgres"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:latest
        ports:
        - containerPort: 5432
          name: postgredb
        env:
          - name: POSTGRES_DB
            value: myapp
          - name: POSTGRES_USER
            value: myuser
          - name: POSTGRES_PASSWORD
            value: passwd
        volumeMounts:
        - name: postgredb
          mountPath: /var/lib/postgresql/data
          subPath: postgres
  volumeClaimTemplates:
  - metadata:
      name: postgredb
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 1Gi

kubectl apply -f ./postgres.yaml

dba9@dba9-minikube2:~/otus/postgres$ minikube service postgres --url -n pg-kub2
http://192.168.49.2:31045
```

#### Устанавливаем клиента
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/postgres.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt update
sudo apt install postgresql-client-14
```

#### Проверяем что все работает
```
psql -h 192.168.49.2 -p 31045 -U myuser -W myapp
Password:
psql (14.9 (Ubuntu 14.9-1.pgdg22.04+1), server 16.0 (Debian 16.0-1.pgdg120+1))
WARNING: psql major version 14, server major version 16.
         Some psql features might not work.
Type "help" for help.

myapp=#
```
