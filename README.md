# Montando um ambiente de produção com Kubernetes

Com essa prática iremos construir 4 máquinas virtuais, sendo uma para hospedar os Rancher e mais 3 para o ambiente de produção.

Fonte dos estudos: [DevOps Ninja: Docker, Kubernetes e Rancher](https://www.udemy.com/course/devops-mao-na-massa-docker-kubernetes-rancher/).

Git referência, do instrutor do curso, [Jonathan Baraldi](https://github.com/jonathanbaraldi/devops).

#### Para a prática, foi necessário ter:
    - 4 máquinas virtuais com 2 nucleos e 4gb RAM
    - 1 domínio
    - S.O. Ubuntu 16.04 LTS
    -  <dev-ops-ninja.com> foi o domínio utilizado pelo instrutor e em diversos casos irei manter para referência.

## Aula 1 -  Introdução
    - Apresentou e descreveu o curso como um todo.

## Aula 2 -  Containers

Explicou:

	- Containers Docker.
	- Registro.
	- Kubernetes.
	- Arquitetura do Rancher e Documentação do kubernetes na documentação oficial do Rancher.

## Aula 3 - DevOps
	- Falou sobre as práticas DevOps, ferramentas e conceitos.

## Aula 4 - Criando o ambiente

	Nesta aula foi verificada a arquitetura do ambiente, instalado o Docker e feito uma revisão.
    O docker foi instalado em todas as maquinas através de um shellscript disponibilizado pelo Rancher.

```sh
$ sudo su
$ curl https://releases.rancher.com/install-docker/19.03.sh | sh
$ usermod -aG docker ubuntu
```
## Aula 5 - Construindo sua aplicação

#### Fazer build das imagens, rodar docker-compose

Nesse exercício foram construidas as imagens que serão usadas em conjunto com o docker-compose. 

Sempre que aparecer `<dockerhub-user>`, será o ponto onde é necessário inserir o usuário cadastrado no dockerhub.

Foram instalados os pacotes Git, Python, Pip e o Docker-compose.
```sh

$ sudo su
$ apt-get install git -y
$ curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
$ ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```
Com os pacotes instalados, baixei o código fonte para fazer as build's e rodar os containers.
```sh
$ cd /home/ubuntu
$ git clone https://github.com/jonathanbaraldi/devops
$ cd devops/exercicios/app
```


#### Container=REDIS

Foi feita a build da imagem do Redis a seguir.
```sh
$ cd redis
$ docker build -t <dockerhub-user>/redis:devops .
$ docker run -d --name redis -p 6379:6379 <dockerhub-user>/redis:devops
$ docker ps
$ docker logs redis
```
Com isso temos o container do Redis rodando na porta 6379.



#### Container=NODE

Foi feita a build do container do NodeJs, que contém a nossa aplicação.
```sh
$ cd ../node
$ docker build -t <dockerhub-user>/node:devops .
```

Foi feito uma ligação do Redis com o node, através das instruções a seguir:

```sh
$ docker run -d --name node -p 8080:8080 --link redis <dockerhub-user>/node:devops
$ docker ps 
$ docker logs node
```
Assim, temos a aplicação rodando conectada ao Redis. A API para verificação pode ser acessada em `/redis`

#### Container=NGINX

Foi feita a build do container do nginx, que será o balanceador de carga.

```sh
$ cd ../nginx
$ docker build -t <dockerhub-user>/nginx:devops .
```

Criando o container do nginx a partir da imagem e fazendo a ligação com o container do Node

```sh
$ docker run -d --name nginx -p 80:80 --link node <dockerhub-user>/nginx:devops
$ docker ps
```

Podemos acessar, então, nossa aplicação nas portas 80 e 8080 no ip da nossa instância.

Iremos acessar a api em /redis para nos certificar que está tudo ok, e depois iremos limpar todos os containers e volumes.

```sh
$ docker rm -f $(docker ps -a -q)
$ docker volume rm $(docker volume ls)
```

#### DOCKER-COMPOSE

O compose utilizado foi criado assim:

```yml
# Versão 2 do Docker-Compose
version: '2'

services:
    
    nginx:
        restart: "always"
        image: <dockerhub-user>/nginx:devops
        ports:
            - "80:80"
        links:
            # Colocar mais nós para escalar
            - node
            # - node-2
            
    redis:
        restart: "always"
        image: <dockerhub-user>/redis:devops
        ports:
            - 6379

    mysql:
        restart: "always"
        image: mysql
        ports:
            - 3306
        environment:
            MYSQL_ROOT_PASSWORD: 123
            MYSQL_DATABASE: books
            MYSQL_USER: apitreinamento
            MYSQL_PASSWORD: 123 
    
# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

    node:
        restart: "always"
        image: <dockerhub-user>/node:devops
        links:
            - redis
            - mysql
        ports:
            - 8080
        volumes:
            -  volumeteste:/tmp/volumeteste
    
# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

# Mapeamento dos volumes
volumes:
    volumeteste:
        external: false


```

É preciso editar o arquivo docker-compose.yml substituindo as partes já citadas anteriormente que se encontram em:

Linha 8 = `<dockerhub-user>/nginx:devops`

Linha 18 = `image: <dockerhub-user>/redis:devops`

Linha 37 = `image: <dockerhub-user>/node:devops`

Após alterado, subindo utilizando `up -d`, subiremos a stack toda.

```sh
$ cd ..
$ vi docker-compose.yml
$ docker-compose -f docker-compose.yml up -d
$ curl <ip>:80 
	----------------------------------
	This page has been viewed 29 times
	----------------------------------
```

Acessando a máquina através do IP na porta 80, podemos olhar os logs através do docker logs e fazer o carregamento do banco em /load. Para terminar nossa aplicação temos que rodar o comando do docker-compose abaixo:

```sh
$ docker-compose down
```

# Aula 6 - Rancher - Single Node

### Instalar Rancher - Single Node

Nesse exercício iremos instalar o Rancher 2.2.5 versão single node, onde o rancher e seus componentes ficam em um único container.

Ao entrar no host responsável por hospedar o Rancher server, verificamos se há algum container e seu status. Após, faremos a instalação do Rancher:

```sh
    $ docker ps -a
    $ docker run -d --name rancher --restart=unless-stopped -v /opt/rancher:/var/lib/rancher  -p 80:80 -p 443:443 rancher/rancher:v2.4.3
```

Foi criado um volume de persistência `-v /opt/rancher:/var/` para que, em caso de desligamento, queda ou quebra do serviço, as informações persistam e possam ser acessadas pela nova máquina que irá subir.
Com o Rancher rodando, adicionou a entrada de DNS para o IP de cada máquina.

```sh
    $ rancher.<dominio> = IP do host A
```

# Aula 7 - Kubernetes

### Criar cluster Kubernetes

Nesse exercício foi criado um cluster Kubernetes, onde foi instalado o kubectl no host A, que será usado para interagir com o cluster. O cluster deve ser criado pelo painel de administração do rancher, que pode ser acessado pelo endereço pré-configurado anteriormente. A criação do cluster é facil, no painel global, clicar em "Add Cluster" > "From existing nodes (custom)" > nomear o cluster > configurar o cluster como preferir > Next > Copiar comando gerado, como o demonstrado a seguir:

Após fazer a configuração (lembrando de selecionar as caixas "etcd", "control plane" e "worker"), o Rancher irá exibir um comando de docker run, para adicionar os host's. Adicionar o host B e host C.

O comando a seguir é semelhante ao que será gerado no painel do Rancher.

```sh
    $ docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.4.3 --server https://rancher.dev-ops-ninja.com --token 8xf5r2ttrvvqcxdhwsbx9cvb7s9wgwdmgfbmzr4mt7smjbg4jgj292 --ca-checksum 61ac25d1c389b26c5c9acd98a1c167dbfb394c6c1c3019d855901704d8bae282 --node-name k8s-1 --etcd --controlplane --worker
```

Esse cluster terá 3 nós e essa configuração deve ser feita em cada um dos nós, lembrando apenas de mudar o `--node-name` para cada máquina.
