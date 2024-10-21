# Configuração de Infraestrutura AWS com Terraform

Este repositório contém a configuração Terraform para criar uma infraestrutura básica na **AWS**, **incluindo uma VPC**, **Subnet**, **Grupo de Segurança**, **Key Pair** e **uma instância EC2** com o servidor **Nginx** instalado.

Desafio Online da Vespenses feito por Fabrícia Fernandes, com 💙

## 🖇️ Índice
- <a href="#requisitos">Requisitos</a>
- <a href="#descricao">Descrição Técnica</a>
- <a href="#melhorias">Melhorias de Segurança</a>
- <a href="#resumo">Breve Resumo</a>
- <a href="#modificacoes">Modificações Realizadas</a>
- <a href="#autora">Pessoa Autora</a>

## Requisitos

- **Terraform** versão >= 1.9.0
- Credenciais de acesso à **AWS** com permissões para criar os recursos: VPC, EC2, CloudWatch, KMS, etc.
- Configuração para a região **us-east-1** da AWS.

## 📋 Descrição Técnica

- **Provider**: AWS na região `us-east-1`.
- **Variáveis**: 
  - `projeto`: Nome do projeto (default: "VExpenses").
  - `candidato`: Nome do candidato (default: "SeuNome").
- **Chave Privada**: Uma chave RSA gerada para uso com a instância EC2.
- **VPC**: Uma VPC com bloco CIDR `10.0.0.0/16`, suporte a DNS habilitado.
- **Sub-rede**: Uma sub-rede na VPC com bloco CIDR `10.0.1.0/24`.
- **Gateway de Internet**: Permite comunicação da VPC com a internet.
- **Tabela de Rotas**: Define rotas para tráfego através do gateway de internet.
- **Grupo de Segurança**: 
  - Permite acesso SSH (porta 22) restrito a um IP específico.
  - Permite todo o tráfego de saída.
- **AMI**: A imagem mais recente do Debian 12.
- **Instância EC2**: Instância `t2.micro` associada à sub-rede e ao grupo de segurança, com um script de inicialização que instala e inicia o Nginx.
- **Saídas**: A chave privada e o IP público da instância EC2.

O arquivo `main.tf` define a seguinte infraestrutura com as seguintes notas:

```hcl
<!-- #Bloco Terraforme -->
terraform {
  required_version = ">= 1.9.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.72.1"
    }
  }
}
<!-- Específica a versão mínima necessária do Terraform e a versão do provedor, isso garante compatibilidade com os recursos mais recentes, é importante para evitar incompatibilidades de versões futuras ou recursos obsoletos. -->

<!-- #Configurações do AWS Provider: -->
provider "aws" {
  region = "us-east-1"
}
<!-- Aqui estamos especificando que vamos usar a AWS como nosso provedor de nuvem. Estamos escolhendo a região onde nossa infraestrutura será criada. Neste caso, é a região "us-east-1". -->

<!-- #Variáveis -->
variable "projeto" {
  description = "Desafio Devops da Vexpenses"
  type        = string
  default     = "VExpenses"
}
<!-- Esta variável armazena o nome do projeto, que é do tipo string. Caso não seja fornecido um valor, o valor padrão será "VExpenses". -->

variable "candidato" {
  description = "Fabricia Fernandes"
  type        = string
  default     = "Fabricia"
}
<!-- Esta variável armazena o nome do candidato, que é do tipo string. Caso não seja fornecido um valor, o valor padrão "SeuNome". Essas variáveis ajudam a personalizar os nomes dos recursos que serão criados. -->

<!-- #Chave Privada -->
resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}
<!-- Estamos criando uma chave privada. O algoritmo é o RSA e a chave terá 2048 bits, que é um tamanho seguro para chaves. -->

<!-- #Par de Chaves AWS para instância EC2 -->
resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}
<!-- Cria um par de chaves para acesso à instância EC2, onde o Key_name  é o nome da chave e é composto pelas variáveis projeto e candidato, e public_key é a chave pública gerada a partir da chave privada. -->

<!-- #VPC (Virtual Private Cloud) -->
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}
<!-- Cria uma rede virtual isolada (VPC), na AWS com as seguintes definições:
cidr_block = "10.0.0.0/16": Define um bloco de endereços IP para a VPC.
enable_dns_support: Habilita o suporte a DNS.
enable_dns_hostnames: Permite que instâncias na VPC recebam nomes DNS.
tags: Adiciona uma tag com o nome da VPC. -->

<!-- #Sub-rede: Subnet -->
resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}
<!-- Cria uma sub-rede dentro da VPC:
vpc_id: Associa a sub-rede à VPC criada anteriormente.
cidr_block = "10.0.1.0/24": Define um bloco de IPs para a sub-rede.
availability_zone: Especifica a zona de disponibilidade onde a sub-rede será criada. -->

<!-- #Gateway de Internet -->
resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}
<!-- Cria um gateway para permitir que a VPC se comunique com a internet, associando o gateway à VPC criada. -->

<!-- #Tabela de Rotas -->
resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}
<!-- Cria uma tabela de rotas para a VPC,  onde route define uma rota que permite tráfego para qualquer endereço IP (0.0.0.0/0) através do gateway de internet e tags adiciona uma tag com o nome da tabela de rotas. -->

<!-- #Associação da Tabela de Rotas -->
resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id
}
<!-- Associa a tabela de rotas à sub-rede criada:
subnet_id: Referencia a sub-rede.
route_table_id: Referencia a tabela de rotas. -->

<!-- #Grupo de Segurança para instâncias EC2 -->
resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir acesso na porta 22"
  vpc_id      = aws_vpc.main_vpc.id

   <!-- # Regras de entrada -->
  ingress {
    description = "Permitir SSH de IP confiavel"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["203.0.133.1/32"] <!--#Deve ser substituído por um IP real do usuário. -->

  }

  ingress {
    description = "Permitir trafego HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Permitir trafego HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  <!-- # Regras de saída -->
  egress {
    description = "Permitir todo o trafego de saida"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}
<!-- Cria um grupo de segurança para controlar o tráfego da instância EC2 onde ingress define regras de entrada e egress define regras de saída. 
E o from_port / to_port indica que as conexões de entrada ao serviço de SSH ocorrerão nas portas definidas. -->

<!-- #Busca a AMI (Amazon Machine Image) do Debian 12 -->
data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}
<!-- Busca a imagem mais recente do Debian, e o filter, como seu nome já diz, ele  filtra as imagens baseando-se em critérios como nome e tipo de virtualização. -->

<!-- #Instância EC2 com Nginx instalado -->
resource "aws_instance" "debian_ec2" {
  ami                         = data.aws_ami.debian12.id
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.main_subnet.id
  key_name                    = aws_key_pair.ec2_key_pair.key_name
  security_groups             = [aws_security_group.main_sg.name]
  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    encrypted             = true
    delete_on_termination = true
  }

  	<!-- #Script de inicialização do Nginx -->
  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              apt-get install ngnix -y
              systemctl start nginx
              systemctl enable nginx
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}
<!-- Cria uma instância EC2, com as seguintes características:
ami: A imagem a ser usada, que é a imagem Debian 12.
instance_type = "t2.micro": Tipo de instância;
subnet_id: A sub-rede em que a instância será criada;
key_name: A chave que será usada para acessar a instância;
security_groups: O grupo de segurança associado à instância;
associate_public_ip_address: A instância terá um IP público;
root_block_device: Define as configurações do disco da instância;
volume_size = 20: O tamanho do disco (20 GB);
delete_on_termination: O disco será deletado quando a instância for encerrada;
user_data: Script que será executado na inicialização da instância, atualizando o sistema;
tags: Adiciona uma tag com o nome da instância. -->

<!-- #Grupo de Logs para VPC e criptografía de Logs -->
resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
  name              = "/aws/vpc/flow-logs-${var.projeto}"
  retention_in_days = 90
  kms_key_id        = aws_kms_key.cloudwatch_kms_key.arn
}
resource "aws_kms_key" "cloudwatch_kms_key" {
  description             = "KMS key para criptografia dos logs de fluxo da VPC"
  deletion_window_in_days = 10
}

resource "aws_flow_log" "vpc_flow_log" {
  vpc_id               = aws_vpc.main_vpc.id
  log_destination_type = "cloud-watch-logs"
  log_destination      = aws_cloudwatch_log_group.vpc_flow_logs.arn
  traffic_type         = "ALL"
}
<!-- Foi adicionado um grupo de logs no CloudWatch com criptografia KMS para os logs de tráfego da VPC, aumentando a segurança e conformidade. A criptografia da root block device já está ativada, para garantir a segurança dos dados. Lembrando que todos os volumes de dados também devem ser criptografados.
O código já tem a automação para instalar o Nginx, está funcional, mas uma melhoria possível seria incluir um monitoramento de saúde para verificar se o Nginx está funcionando após o provisionamento. Se for necessário ter um monitoramento mais detalhado das instâncias EC2, pode-se habilitar o CloudWatch Detailed Monitoring, que coleta dados mais frequentemente (a cada minuto). -->

<!-- #Saídas -->
output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
<!-- Exibe a chave privada gerada para acesso à instância. E o segundo output exibe o IP público da instância, para que possa se conectar a ela. -->
```

### ⚙️ Melhorias de Segurança

- **Acesso SSH**: A regra de entrada do grupo de segurança foi modificada para permitir SSH apenas de um IP específico, aumentando a segurança da instância.

## 📄 Breve Resumo

Esse código configura uma infraestrutura básica na **AWS** com uma **VPC**, **subnet**, **instância EC2**, **grupo de segurança** e um **par de chaves**. 

Ele é estruturado de forma que possa facilmente modificar os valores de variáveis e adicionar novos recursos conforme necessário. 

É uma maneira eficiente de criar e gerenciar infraestrutura na nuvem, permitindo que se concentre no desenvolvimento e na operação de suas aplicações.

## 🔧 Modificações Realizadas

Este Código teve como melhorias, a segurança aumentada, o acesso via **SSH** estará restrito a um **IP confiável**, e as regras de ingress serão mais rígidas, a criptografia de logs e volumes protege os dados de forma eficaz. 

A eliminação de **log_group_name** para **log_destination** segue as recomendações mais recentes da AWS, evitando futuros problemas de depreciação. 

O **Nginx** será instalado automaticamente, e há uma verificação para garantir que ele esteja funcionando corretamente.

O monitoramento detalhado do CloudWatch coleta dados mais frequentemente, melhorando a visibilidade do estado da instância.

Essas melhorias resultam em um ambiente mais seguro, otimizado e em conformidade com as práticas mais recentes da AWS.

## ✒️  Pessoa Autora

<img style="width:150px" src="image/eu.png" alt="Imagem de desenvolvedora"> 

Fabrícia Fernande

[Github](https://github.com/Fabriciabli)
[Linkedin](https://www.linkedin.com/in/fabriciafernandes/)
