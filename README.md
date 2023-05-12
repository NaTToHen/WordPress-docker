# Instância ec2 linux com docker WordPress

  O projeto consiste no deployment de uma aplicação WordPress dentro de um ambiente LoadBalance com Auto Scaling e banco de dados MySql RDS.
  
## Intâncias e ambiente linux/docker
Cada intância é inicializada a partir de uma AMI pré configurada.
- user_data:
```
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
-
