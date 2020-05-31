# PROJETO INTEGRADOR 

### SITUAÇÃO: 
Recebimento de uma aplicação em Node.js em desenvolvimento, com funcionalidades a serem passadas por teste de qualidade.

### PROJETO DA EQUIPE LOVELACE: 
Criar uma pipeline de entrega contínua da aplicação a ser armazenada na Cloud.

### TECNOLOGIAS UTILIZADAS: 
Ansible, Docker, Jenkins, AWS.

## ARQUITETURA:

![Imagem Projeto](https://user-images.githubusercontent.com/60946367/83334657-5989bf00-a27e-11ea-8ff4-2cef9ff5f284.png)
[](url)

### Primeiros passos:
#### 1. Configurar o ambiente para executar o plano de ação:
- Instalar a VirtualBox;
- Dentro da VM, instalar os softwares VSCode, Ansible, Docker e Jenkins;
- Criar contas na AWS.
#### 2. No desktop da máquina virtual, criar um arquivo para as senhas no VSCode ou no próprio terminal:
$ vi ~/.ansible/.vault_pass
#### 3. Criar o arquivo aws_credentials.yml na pasta vars do repositório do projeto com as credenciais de segurança do usuário root da AWS:
- Acessar o console da AWS com o user root e seguir o seguinte caminho: My Security Credentials > Access keys (access key ID and secret access key) > Create New Access Key;
- Colocar o AWSAccessKeyId e o AWSSecretKey em aws_credentials.yml;
- Copiar as informações do arquivo para futuro acesso;
- Encriptar aws_credentials.yml:
$ ansible-vault encrypt playbooks/vars/aws_credentials.yml
#### 4. Criar um repositório para o projeto no Github e criar o arquivo .gitignore, que não permitirá que determinados arquivos sejam versionados no repositório (como aws_credentials.yml, por exemplo);
#### 5. Configurar AWSCLI no desktop da VM;
#### 6. Configurar as credenciais do usuário da AWS no desktop e colocar a região de trabalho (no momento, compensa financeiramente utilizar a us-east-1):
$ aws configure
$ aws iam list-access-keys
$ aws ec2 describe-regions
 
## EXECUÇÃO DO PROJETO: 
#### 1. Customizar os Playbooks:
- Configurar a infraestrutura em aws_provisioning.yml (arquivo principal de configurações), aws_provisioning_vpc.yml, aws_provisioning_jenkins.yml, aws_provisioning_producao.yml e aws_provisioning_homolog.yml;
- Em aws.yml, configurar:
AWS_access_key: "{{ AWSAccessKeyId }}"* > Credenciais AWS (está no arquivo criptografado vars/aws_credentials.yml).
AWS_secret_key:"{{ AWSSecretKey }}"* > Credenciais AWS (está no arquivo criptografado vars/aws_credentials.yml).
instance_type (tipo da instância que será criada na AWS): t2.micro;
security_group (nome do Security Group): "{{ projeto }}-ws-sg";
image_ami (imagem a ser utilizada para criar a máquina na AWS): ami-07ebfd5b3428b6f4d;
KeyPairDev (nome da key-pair): "{{ projeto }}-key";
region (região a ser usada na AWS): us-east-1;
vpc_cidr_block: 10.0.0.0/16;
cidr_block: 10.0.1.16/28;
vpc_name: "{{ projeto }}-vpc";
route_tag (Nome da Route Table, que é parte da VPC): "{{ projeto }}-route";
sub_tag (Nome da Subnet, que é parte da VPC): "{{ projeto }}-subnet";             
- Criar a Key Pair das instâncias EC2; 
- Criar 3 Amazon EC2 (Amazon Elastic Compute Cloud, uma máquina virtual na nuvem) em cada playbook referente ao respectivo ambiente: aws_provisioning_jenkins.yml (a EC2 criada será o servidor Jenkins), aws_provisioning_producao.yml e aws_provisioning_homolog.yml;
- Em aws_provisioning_jenkins.yml, criar as seguintes tasks:
Launch the new EC2 Instance 22 (Executar a nova instância EC2 de acordo com as variáveis do itens vars_files e o vars);
Wait for SSH to come up (Aguardar o SSH chegar);
Create AWS ECR (Criar a Amazon Elastic Container Registry, um tipo de repositório que armazena e gerencia as imagens de contêineres do Docker geradas pela pipeline).
- Em aws_provisioning_homolog.yml e aws_provisioning_producao.yml, criar estas tasks:
Launch the new EC2 Instance 22;
Wait for the SSH to come up;
Create iam user “{{ name_aim_user_s3 }}” (Criar o usuário IAM);
Create a bucket (Criar um bucket, recurso de armazenamento em cloud da AWS);
Create S3 (Denomina o bucket criado como Simple Storage Service, que armazenará os dados gerados pelo ambiente - seu nome deve ser um nome único em letras minúsculas com tamanho entre 3 e 63 caracteres).
- config_all-ec2.yml deve atualizar os pacotes em todas as EC2 e instalar softwares e suas dependências, como awscli, java, python;
- install_docker_all-ec2.yml deve instalar o docker nas EC2 que foram criadas;
- install_ansible_ec2-jenkins.yml deve Instalar o Ansible e pacotes para facilitar configurações;
- install_jenkins_ec2-jenkins.yml deve instalar e configurar o Jenkins na EC2 master.
 
#### 2. Provisionar EC2, S3, IAM e ECR na AWS
- Criar uma VPC (Virtual Private Cloud) e outros itens importantes para sua utilização (como Subnet e Security Group):
- Gerar uma key pair e salvá-la;
- Criar 3 EC2 (uma para o Jenkins, uma para o ambiente de Homologação e outra para o ambiente de Produção), um ECR (Elastic Container Registry, que tem a função de armazenar e gerenciar as imagens de contêineres do Docker geradas pela pipeline), um user e um bucket S3; 
$ ansible-playbook playbooks/aws_provisioning.yml
- Verificar se a VPC foi criada corretamente:
	$ ansible-inventory --graph aws_ec2
 
#### 3. Configurar máquinas EC2
- Atualizar pacotes e instalar AWCCli, Java e Python nas 3 EC2:
$ ansible-playbook playbooks/config_all-ec2.yml
- Instalar o Docker nas 3 EC2:
$ ansible-playbook playbooks/install_docker_all-ec2.yml
- Instalar o Ansible nas 3 EC2:
$ ansible-playbook playbooks/install_ansible_ec2-jenkins.yml
- Instalar o Jenkins apenas na EC2 master:
$ ansible-playbook playbooks/install_jenkins_ec2-jenkins.yml
 
#### 4. Configurar o Jenkins
- Entrar no terminal e acessar a url de acordo com o nome ou IP público gerado na AWS;
- Logar no Jenkins com usuário e senha definidos no playbook install_jenkins_ec2-jenkins.yml;
- Seguir o caminho:
Gerenciar Jenkins > Gerenciador de Plugins > Disponíveis > Buscar e selecionar: Github, Pipeline, Docker pipeline, AWS Code Pipeline, SSH, SSH Agent, SSH Build Agent, SSH Credentials;
- Acessar a EC2 via SSH:
ssh -i @ ssh -i .pem ubuntu@
- Verificar o usuário Jenkins:
$ id jenkins
- Incluir o usuário Jenkins no grupo Docker do Linux:
$ sudo addgroup jenkins docker
- Parar o serviço do Jenkins:
$ sudo service jenkins stop
- Subir o serviço do Jenkins:
$ sudo service jenkins start
- Validar o serviço do Jenkins:
$ sudo service jenkins status
- Configurar nodes Homolog e Produção para a execução do pipeline:
Gerenciar Jenkins > Gerenciar nós > novo nó
- Adicionar Credenciais para acessar as instâncias:
Credentials > Add Credentials (Global, SSH username e private key)
- Adicionar Credenciais para acessar AWS ECR:
Credentials > Add Credentials (AWS Credentials, Global, ID e Private Key AWS)

#### 5. Pipeline Jenkins
Com a aplicação pronta para release, é possível criar um Job, cuja função será realizar o deploy de forma automática.
 
 
Aplicação Node.JS utilizada
https://github.com/bgsouza/digitalhouse-devops-app
 
 
### EVIDÊNCIAS

#### PIPELINE:

##### Pipeline Homologação executado com sucesso:
![HOMOLOG](https://user-images.githubusercontent.com/60946367/83364533-44915680-a378-11ea-8828-0bed5143f3a4.jpg)

##### Pipeline Produção executado com sucesso:
![Produção](https://user-images.githubusercontent.com/60946367/83364636-00eb1c80-a379-11ea-8ac4-207d43557d70.png)


#### URL:

#### HealthCheck Homologação executado com sucesso:

#### HealthCheck Produção executado com sucesso:

#### Upload Homologação executado com sucesso:

#### Upload Produção executado com sucesso:


#### AWS S3

##### Buckets S3 criados:

- Bucket S3 Homolog com a imagem:

- Bucket S3 Produção com a imagem:
 
 
### REFERÊNCIAS:

#### Professores:

Bruno G. Souza - https://github.com/bgsouza/digitalhouse-devops-app

Krishna Pennacchioni - https://github.com/agentelinux/devops-pi/tree/grupo1

#### Material do curso:

Playground Digital House

#### Documentação Oficial:

https://www.ansible.com/

https://galaxy.ansible.com/

https://github.com/

https://www.docker.com/

https://www.jenkins.io/

https://aws.amazon.com/pt/

https://www.markdownguide.org/
 
