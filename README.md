# Desafio 1


crie o Dockerfile
execute 
```
docker build -t <seu_nome_usuario_dockerhub>/conversaotemperatura:v1
```
crie o .dockerignore
```
*node_modules*
```

```
docker login
docker push <seu_nome_usuario_dockerhub>/conversao-temperatura:v1
docker tag <seu_nome_usuario_dockerhub>/conversao-temperatura:v1 <seu_nome_usuario_dockerhub>/conversao-temperatura:latest
docker push <seu_nome_usuario_dockerhub>/conversao-temperatura:latest
```