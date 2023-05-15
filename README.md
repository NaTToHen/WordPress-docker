# Instância ec2 linux com docker WordPress

  O projeto consiste no deployment de uma aplicação WordPress dentro de um ambiente LoadBalance com Auto Scaling e banco de dados MySql RDS.
  
## Intâncias e ambiente linux/docker
Cada intância é inicializada a partir de uma AMI pré configurada.
- **tipo da instância:** t2.micro: 1 cpu, 1gb ram, 8gb gp3 ssd
- **user_data da instância:**
```shell
#!/bin/bash
yum update
yum install -y amazon-efs-utils
yum -y install docker
service docker start
usermod -a -G docker ec2-user
chkconfig docker on
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
reboot
```
**docker e docker-compose**

O serviço WordPress é implementado em um container docker. O container foi criado a partir de um docker-compose facilitando a sua inicialização.
**Doker-compose.yml** utilizado: 

```yaml
version: "3"
services:
  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_HOST: <endpoint do bd>
      MYSQL_ROOT_PASSWORD: senha do banco
      MYSQL_DATABASE: nome do banco
      MYSQL_USER: user do banco
      MYSQL_PASSWORD: senha do usuario
  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: user do banco
      WORDPRESS_DB_PASSWORD: senha do banco
      WORDPRESS_DB_NAME: nome do banco
    volumes:
      ["mnt/docker/docker:/var/www/html"]
volumes:
  mysql: {}
```

O arquivo do docker-compose executa as seguintes etapas:

- Criação de um container MySql e conecção com o banco de dados RDS
- Criação do container WordPress
- Espelhamento dos sistemas de arquivo

## EFS (Elastic File System)
O EFS (Elastic File System) é um serviço de armazenamento em nuvem gerenciado pela Amazon Web Services que fornece um sistema de arquivos compartilhado e escalável para uso com instâncias do Amazon Elastic Compute Cloud (EC2). Dentro do EFS são armazenados todos os arquivos do serviço WordPress. **O EFS também é montado dentro do container WordPress**.

**Montagem do EFS**

Após a criação do EFS e de seu ponto de acesso `mnt/docker`, ele é automaticamente montado em cada instância ec2 necessária utilizando o comando:

```shell
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <id do efs>:/ mnt/docker
```

Para a montagem automática após reinicio foi inserida uma entrada no arquivo: `/etc/fstab`.

```
<dns do efs>:/ mnt/docker/ nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport 0 0
```


## VPC e Security Groups

A VPC possui dois ranges de CIDR diferentes  


10.1.96.0/24: subnets públicas  
10.1.0.0/24: subnets privadas


O VPC foi criado  com duas subnets privadas e duas subnets públicas, uma em cada Zona de Disponibilidade.  
- Foi criado e associado um Internet Gateway à rota publica para liberar comunicação com a Internet
- Foi criado um NAT Gateway com um Elastic IP alocado, e associado á rota privada, associando também a uma subnet publica a fim de garantir tráfego de Internet às subnets privadas.


Em Route tables:
- Foram criadas duas Route Tables, uma pública e uma privada.
- Na Route Table pública foi criada uma rota para a Internet (0.0.0.0/0) associada ao Internet Gateway. Foram associadas a ela as duas subnets públicas.
- Na Route Table privada foi criada uma rota para a Internet (0.0.0.0/0) associada ao NAT Gateway. Foram associadas a ela as duas subnets privadas.


**Security Groups**  
Optei por utilizar somente um Security Group abrindo as seguintes portas:  
| serviço |  | porta | ip liberado |
|--- |--- |--- |--- |
| MYSQL/Aurora | TCP | 3306 | 0.0.0.0/0 |
| HTTPS | TCP | 443 |	0.0.0.0/0 |
| HTTP | TCP | 80 | 0.0.0.0/0 |
| NFS | TCP | 2049 | 0.0.0.0/0 |
| SSH	| TCP	| 22 | 0.0.0.0/0 |

## RDS (Relational Database Service)

- Configuração do banco de dados MySql:

  - instância db.t3.micro
  - MySql
  - ram: 1 GB
  - Armazenamento: 20 GiB (gp2)

