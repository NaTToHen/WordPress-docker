# Instância ec2 linux com docker WordPress

  O projeto consiste no deployment de uma aplicação WordPress dentro de um ambiente LoadBalance com Auto Scaling e banco de dados MySql RDS.
  
# Arquitetura
  
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
- **EFS:** O EFS (Elastic File System) é um serviço de armazenamento em nuvem gerenciado pela Amazon Web Services que fornece um sistema de arquivos compartilhado e escalável para uso com instâncias do Amazon Elastic Compute Cloud (EC2). Dentro do EFS são armazenados todos os arquivos do serviço WordPress. **O EFS também é montado dentro do container WordPress**.

**Montagem do EFS**

Após a criação do EFS e de seu ponto de acesso `mnt/docker`, ele é automaticamente montado em cada instância ec2 necessária utilizando o comando:

```shell
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <id do efs>:/ mnt/docker
```

Para a montagem automática após reinicio foi inserida uma entrada no arquivo: `/etc/fstab`.

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




