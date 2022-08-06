# Desafio 2
 08 - 2022

Instale o k3d - https://k3d.io/v5.4.4/

configure o cluster 

k3d cluster create

sem o load-balance

k3d cluster create --no-lb


Crie um cluster maior porem cuidado com os recursos da sua maquina

k3d cluster create meucluster --servers 3 --agents 3


# Kubernetes

## Pod
Menor objeto do cluster kubernetes
aqui cria e executas os containers
tem 1 ou mais contsainers
compartilha o os mesmos recursos de rede e pode ter filesystem compartilhado
Coloque os container em pods separados para manter a resiliencia e poder escalar as aplicações individualmente

pode usar side-car que normalmente é usado com um container auxiliar associado a sua aplicação principal.eX: um agente enviando os logs da sua api para um servidor de logs.


-> crie o manifesto para aplicar no cluster. Ele trabalha de forma declarativa e fará exatamente o que foi colocado no manifesto

ele é um arquivo yaml

os campos princiupais são:
apiVersion: -> grupo de apis que o tipo de objeto que está sendo criado utiliza. Execute no terminal: kubectl api-resources ou ubectl api-resources | grep pod
kind: -> tipo de objeto
metadata: -> metadados do objeto.Ex: nome, labels, annotations, etc...
spec: -> especificações do objeto

k8s/pod.yaml
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: meupod
spec: 
  containers:
    - name: web
      image: fabricioveronez/web-page:blue
      ports:
        - containerPort: 80
```

```
kubectl apply -f k8s/pod.yaml

kubectl get po
kubectl get po -o wide
kubectl describe pod meupod
```

Para acessar use o kubectl port-forward

```
kubectl port-forward pod/meupod 8081:80
```

Somente o pod não tem escalabilidade, resiliencia nem mesmo troca de versão... simplesmente executa os containers

para garantir isso precisa de um controlador que é o Replicaset



### Conceito de labels e selectors
os objetos são marcados por labels. Esses são elementos de chave e valor para identificar este objeto e a que este objeto faz parte.

```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: meupod
spec: 
  containers:
    - name: web
      image: fabricioveronez/web-page:blue
      ports:
        - containerPort: 80

---

apiVersion: v1
kind: Pod
metadata: 
  name: meupod2
  labels:
    app: green
spec: 
  containers:
    - name: web
      image: fabricioveronez/web-page:green
      ports:
        - containerPort: 80

```

```
kubectl get pods

NAME      READY   STATUS    RESTARTS   AGE
meupod    1/1     Running   0          41m
meupod2   1/1     Running   0          13s

kubectl get pods

NAME      READY   STATUS    RESTARTS   AGE
meupod    1/1     Running   0          42m
meupod2   1/1     Running   0          63s

kubectl get pods -l app=green

NAME      READY   STATUS    RESTARTS   AGE
meupod2   1/1     Running   0          71s


kubectl delete -f k8s/pod.yaml  

```




## Replicaset

No replicaset define o template do pod e tambem as replicas que devem estar ativas.

== Ele garante que a quantidade de replicas desejadas deve ser a quantidade corrente. ==

replicas desejadas == replicas correntes

replicaset.yaml
```
apiVersion: apps/v1
kind: ReplicaSet
metadata: 
  name: meureplicaset
spec: 
  replicas: 10
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec: 
      containers:
        - name: web
          image: fabricioveronez/web-page:blue
          ports:
            - containerPort: 80

```

-->> do metadata para baixo é igual do pod.yaml

```
 kubectl get replicasets
 kubectl get pods
 kubectl describe pod meureplicaset-5mfwx
```

Para conferir a resiliencia:

```
kubectl delete pod meureplicaset-5mfwx
```

ele deve recriar o pod

verificar escalabilidade


mudar especificação do replicaset acrescentas as replicas

```
...
spec: 
  replicas: 10
  selector:
...
```

falta garantis a troca de versão com downtime 0

e para isso é usado o Deployment

## Deployment

Este controlador fica acima do replicaset garantindo a troca de versão.
ele gerencia o replicaset.
Se houver alguma alteração no replicaset será criado novo replicaset que fara a troca dos pods gradativamente. O replicaset antigo permanece para um possivel rolback.

criando um Deployment.

o mesmo manifesto do replicaset com as seguintes alterações:

```
...
kind: Deployment
metadata: 
  name: meudeployment
spec: 
...
```


para fazer o rollback

```
kubectl rollout history deployment meudeployment

deployment.apps/meudeployment 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

kubectl rollout undo deployment meudeployment   

deployment.apps/meudeployment rolled back
```


Agora uma forma de acessar os pods sem o port-forward


Para trabalhar com service, usa-se

Tipos

ClusterIp -> apenas os pods tem acesso 
NodePort -> expoe os pods externament, porem utiliza uma porta, dentro do rabnge 30000-30767 que será exposto em todos os nós do cluster K8S. Muito usado em cenários on-promiese
LocaBalancer -> expoe porem utilixza o serviço do cloudprovider para criar um Loadbalance gerando um IP externo para acessar. podfe usar o MetalLB para criar um ip em cenário Baremetal


Primeiro vamos resolver o problema para que possamos acessar nossos container com apenas uma porta especifica. Issó é feito fazendo um buind de uma porta entre 30000 e 32767 com uma das maquinas do cluster e como estamos usando um cluster baseado em container, vamos fazer o bind usando as tcnicas do bind de portas do docker. O container escolhido é o loadbalancer.

Vamos apagar o cluster e recira-lo

```
k3d delete cluster meucluster

k3d cluster create meucluster --servers 3 --agents 3 -p "30000:30000@loadbalancer"

#para verificar
docker container ls 
```

agora no service, deve colocar a especificação do NodePort expondo a porra 30000 (neste caso)


```yaml
  - protocol: TCP
    port: 80
    nodePort: 30000
```

para verificar

```
kubectl get service

NAME          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes    ClusterIP   10.43.0.1     <none>        443/TCP        5m59s
service-web   NodePort    10.43.30.48   <none>        80:30000/TCP   13s
```

e para acessar no navegador

```
no  navegador

http://localhost:30000
```

para testar mude o container e verifique que as trocas serão feitas gradativamente e sem downtime

dando rollback

```
kubectl rollout undo deployment meudeployment
```



# KUBE-NEWS

Criar o Dockerfile

```Dockerfile
FROM node:16.15.0
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]
```

```
docker build -t hernanisoares/kube-news:v1 .
docker tag hernanisoares/kube-news:v1 hernanisoares/kube-news:latest
docker push hernanisoares/kube-news:v1 
docker push hernanisoares/kube-news:latest
```

crie o manifesto deployment.yaml em ./K8S

```yaml
# Deployment do Postgree

apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgre
spec:
  selector:
    matchLabels:
      app: postgre
  template:
    metadata:
      labels:
        app: postgre
    spec:
      containers:
      - name: postgre
        image: postgres:alpine3.16
        env:
        - name: POSTGRES_PASSWORD
          value: "myPass"
        - name: POSTGRES_USER
          value: "myUser"
        - name: POSTGRES_DB
          value: "kubenews"
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 5432

---

apiVersion: v1
kind: Service
metadata:
  name: postgre
spec:
  selector:
    app: postgre
  ports:
  - port: 5432 ## porta usada pelo service
    targetPort: 5432 ## porta do pod

--- 

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubenews
spec:
  selector:
    matchLabels:
      app: kubenews
  template:
    metadata:
      labels:
        app: kubenews
    spec:
      containers:
      - name: kubenews
        image: hernanisoares/kube-news:v1
        env:
        - name: DB_DATABASE
          value: "kubenews"
        - name: DB_USERNAME
          value: "myUser"
        - name: DB_PASSWORD
          value: "myPass"
        - name: DB_HOST
          value: "postgre"
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: kubenews
spec:
  selector:
    app: kubenews
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30000
  type: NodePort


```

aplique no cluster
```
kubectl apply -f k8s/deployment.yaml 
```





para testar, altere algo no projeto. como teste ser'alterado no arquivo kube-news/src/views/partial/nav-bar.ejs, a linha 5

```
<a class="navbar-brand text-warning" href="/">KubeNews - V2</a>
```

aplique no cluster
```
docker build -t hernanisoares/kube-news:v2 ./src
docker tag hernanisoares/kube-news:v2 hernanisoares/kube-news:latest
docker push hernanisoares/kube-news:v2 
docker push hernanisoares/kube-news:latest

```

No manifesto deployment.yaml altere:

```
    spec:
      containers:
      - name: kubenews
        image: hernanisoares/kube-news:v2
        env:

```


aplique no cluster
```
kubectl apply -f k8s/deployment.yaml 
```






