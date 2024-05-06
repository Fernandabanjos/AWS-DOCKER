   # Instruções da atividade 
- Instalação e configuração do DOCKER ou CONTAINERD no host EC2; (Ponto adicional para o trabalho utilizar a instalação via script de Start Instance (user_data.sh))
- Efetuar Deploy de uma aplicação Wordpress com: container de aplicação RDS database Mysql
- Configuração da utilização do serviço EFS AWS para estáticos do container de aplicação Wordpress
- Configuração do serviço de Load Balancer AWS para a aplicação Wordpress

# Configuração da VPC:
- Acesse o console AWS e pesquise o serviço VPC
- No menu lateral esquerdo, na seção de Virtual private cloud selecione Your VPCs
- Dentro de Your VPCs clique no botão Create VPC
- Altere as seguintes configurações:
- Em Resources to create selecione VPC and more
- Em Name tag auto-generation coloquei o nome "projeto-docker-vpc"
- Em Number of Availability Zones (AZs) selecione 2
- Em NAT gateways selecione In 1 AZ
- Em VPC endpoints selecione None
- Clique em Create VPC

# Configuração dos Security Groups:
- Acesse o console AWS e entre no serviço EC2
- No menu lateral esquerdo, na seção de Network & Security, selecione Security Groups
- Dentro de Security Groups, clique no botão Create security group
- Crie e configure os seguintes security groups usando a VPC criada anteriormente:

  
   - #### Load Balancer
        | Type | Protocol | Port Range |   Source  |
        |:----:|:--------:|:----------:|:---------:|
        | HTTP | TCP      | 80         | 0.0.0.0/0 |

    - #### EC2 
        | Type | Protocol | Port Range |       Source       |
        |:----:|:--------:|:----------:|:------------------:|
        |  SSH |    TCP   |     22     |      0.0.0.0/0     |
        | HTTP |    TCP   |     80     |      0.0.0.0/0     |
        | NFS  |    TCP   |   2049     |        EFS         |

     - #### RDS
        |     Type     | Protocol | Port Range |        Source       |
        |:------------:|:--------:|:----------:|:-------------------:|
        | MYSQL/Aurora |    TCP   |    3306    |    EC2              |

    - #### EFS 
        | Type | Protocol | Port Range |        Source       |
        |:----:|:--------:|:----------:|:-------------------:|
        | NFS  | TCP      | 2049       |    EC2              |

# Elastic File System
- Acesse o console AWS e pesquise pelo serviço EFS
- Clique em Costumize e defina um nome, nesse projeto foi escolhido "projeto-docker" 
- Selecione a VPC criada anteriormente
- Selecione as subnetes privadas de cada AZ
- Em Security groups selecione o grupo EFS que foi criado

# Relational Database Service:
- Pesquise pelo serviço RDS na console AWS
- Clique em Create Database
- Selecione Criação Padrão
- Escolha MySQL com modelo de Nível Gratuito
- Nas configuração crie um nome para a base e,também, um nome de usuário princial e senha
- Na seção Conectividade, selecione a VPC criada anteriormente
- No campo Existing VPC security groups selecionei o grupo "RDS" que foi criado anteriormente
- Load Balancer:
- Na console AWS pesquise pelo serviço do Load Balancers
- Clique em Application Load Balancer
- Defina um nome e escolha a opção "Voltado para a Internet"
- Em Mapeamento de rede, escolha a VPC criada para o projeto
- Selecione as Sub-redes para cada AZ
- Escolha o grupo de segurança feito para o Load Balancer
- Em listerners e roteamento mantenha o protocolo HTTP com a porta 80 aberta e crie um grupo de destino de "Instâncias"
- Clique em Criar Load Balancer

# Modelo de Execução para a criação das instâncias:
- No painel EC2 pesquise pela opção "Modelos de Excecução" e clique em criar
- Defina um nome para o modelo
- Em Imagens de Aplicação selecione o início rápido e selecione:
- Amazon Linux AWS (2023)
- Tipo de instância: t3.small
- Par de chaves: criei um par de chaves em formato .ppk para acesso ssh via puTTY
- Em configuração de redes, em Sub-redes, manter "Não incluir no modelo de execução"
- Selecione grupo de segurança criado "EC2"
- Informe as tags de recursos: Name, CostCenter e Project
- Em "Detalhes Avançados", na seção "Dados do usuário", insira o script:

```
#!/bin/bash

#Atualizar o sistema
sudo yum update -y

#Instalar o Docker
sudo yum install docker -y

#Habilitar e iniciar o serviço Docker
sudo systemctl enable docker
sudo systemctl start docker

#Adicionar o usuário atual ao grupo docker
sudo usermod -a -G docker ec2-user

#Instalar utilitários do Amazon EFS
sudo yum install amazon-efs-utils -y

#Criar o ponto de montagem do EFS
sudo mkdir /mnt/efs

#Montar o EFS
echo "fs-09574dde3bd9ec12d.efs.us-east-1.amazonaws.com:/ /mnt/efs nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0" >> /etc/fstab
sudo mount -a


#Baixar o Docker Compose
curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

#Criar a configuração do Docker Compose
cat << EOF > /home/ec2-user/docker-compose.yaml
version: "3.8"

services:
  wordpress:
    image: wordpress
    volumes:
      - /mnt/efs/website:/var/www/html
    ports:
      - "80:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: 
      WORDPRESS_DB_USER: 
      WORDPRESS_DB_PASSWORD: 
      WORDPRESS_DB_NAME: 

EOF

#Executar o Docker Compose
sudo chown -R ec2-user:ec2-user /home/ec2-user/docker-compose.yaml
sudo docker-compose -f /home/ec2-user/docker-compose.yaml up -d
```

# Auto Scaling Group
- No painel EC2 pesquise pelo serviço Auto Scaling, e clique em criar grupo de auto scaling
- Defina um nome e selecione o modelo de execução que foi criado
- Selecione a VPC criada para o projeto e as sub-redes
- Clique em "Anexar a um balanceador de carga existente" 
- Escolha o grupo de destino
- Ative as verificações de integridade do Elastic Load Balancing
- Defina a capacidade do grupo, neste projeto foi selecionado "2"
- Em escalabilidade defina a capacidade mínima e a capacidade máxima, também, em capacidade 2
- Revise e crie o Auto Scaling

# ENDPOINT
- No painel VPC na AWS, selecione a opção Endpoint 
- Defina um nome para Endpoint, marque a opção "Endpoint do EC2 Instance Connect"
- Em seguida, selecione a VCP criada e o Grupo de Segurança EC2
- Selecione a subrede que o Endpoint será usado e finalize

# Executando a instância EC2

- Escolha uma das instâncias que o Auto Scaling gerou e faça o acesso via SSH
- Verifique a montagem do EFS com o comando `df-h`
- Foi utlizado o comando `docker ps` para visualizar o contêiner e para acessa-lo foi usado o comando:
 ```
 docker exec -it containerID /bin/bash
```
![Captura de tela 2024-05-06 175906](https://github.com/Fernandabanjos/AWS-DOCKER/assets/142920603/ecbe1020-0e57-45b2-b480-737cb5be62b1)
![Captura de tela 2024-05-06 175926](https://github.com/Fernandabanjos/AWS-DOCKER/assets/142920603/6b182472-b63f-40b8-a4ae-a6f4e8e454f2)

- Essas telas são importantes para coletarmos Container ID para verificarnos a conexão do container com o banco de dados (RDS)
  **No interior do conteiner**
- Atualize os pacotes usando o comando `apt-get update` e instale o MySQL por meio do comando:
   ```
  apt-get install default-mysql-client -y
   ```
- Em seguida, acesse o banco de dados com
   ```
  mysql -h [ENDPOINTDORDS] -u [NomedoUsuárioPrincipal] -p
   ```
  - Digitei a senha do usuário
  - Utilizei o comando `show databases;` para listar os bancos de dados disponíveis
  - Utilizei o comando `use dockerdb;` para selecionar o banco de dados dockerdb
  - Utilizei o comando `show tables;` para listar todas as tabelas criadas dentro do banco de dados dockerdb
    ![Captura de tela 2024-05-06 180758](https://github.com/Fernandabanjos/AWS-DOCKER/assets/142920603/df3bccf5-74b7-4176-8454-fff212391645)

- **Configurando o Link (DNS) dos Balanceadores de Carga no Wordpress:**
- Acessei com o ip da instância através do navegador
- Na tela de instalação do WordPress mantive o idioma padrão e cliquei em Continue.
- Na tela seguinte preenchi os dados para criação de um usuário.
- Cliquei em Install WordPress para finalizar.
- Após a instalação cliquei no painel e em opções e alterei o o endereço do wordpress (URL) e o do site (URL)  para o DNS name do Load Balancer.

- ![Captura de tela 2024-05-06 175205](https://github.com/Fernandabanjos/AWS-DOCKER/assets/142920603/f4f02d4d-11d9-4337-b318-386dab0a7335)



