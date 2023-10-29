# Разворачиваем Postgres в minikube

#### Ставим docker
```
dba9@dba9-minikube2:~$ sudo apt-get install ca-certificates curl gnupg
dba9@dba9-minikube2:~$ sudo install -m 0755 -d /etc/apt/keyrings
dba9@dba9-minikube2:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
dba9@dba9-minikube2:~$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
dba9@dba9-minikube2:~$ echo   "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
dba9@dba9-minikube2:~$ sudo apt-get update
dba9@dba9-minikube2:~$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
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
