# Template de EC2 - Estudos AWS

O template cria um Security Group e, opcionalmente (quando `EnvironmentType = Prod`), uma instância EC2 que faz bootstrap via UserData para clonar um repositório e rodar a aplicação em Docker.

Visão geral
----------
O objetivo do template é fornecer um deploy simples e reproduzível para uma aplicação contêinerizada:

- Sempre cria um Security Group com regras para SSH (22) e para a aplicação (porta 8000).
- Quando `EnvironmentType = Prod`, também cria uma instância EC2 (por padrão `t3.micro`) que:
  - instala Docker e Git (suporta AMIs com `yum` ou `apt`),
  - clona o repositório indicado (`RepoUrl` / `Branch`) em `/opt/juliana-app`,
  - builda a imagem Docker e executa um container nomeado `juliana` expondo `8000:80`.

Parâmetros (resumo)
-------------------
- EnvironmentType (String: `Prod` | `Dev`, Default: `Dev`)  
  - `Prod`: cria EC2 + SG. `Dev`: cria apenas o Security Group.
- KeyName (AWS::EC2::KeyPair::KeyName, Default: "")  
  - Nome do KeyPair para SSH. Se deixado vazio, a instância será criada sem KeyPair (não recomendado para acesso SSH).
- MyIp (String, Default: `0.0.0.0/0`)  
  - CIDR permitido para SSH. Recomenda-se usar `seu_ip/32`.
- VpcId (AWS::EC2::VPC::Id)  
  - VPC onde o Security Group e a EC2 serão criados.
- SubnetId (AWS::EC2::Subnet::Id)  
  - Subnet pública (com auto-assign public IP) para a instância.
- RepoUrl (String, Default: https://github.com/JulianaForbici/docker-aws)  
  - Repositório que contém o Dockerfile / código da aplicação.
- Branch (String, Default: `main`)  
  - Branch a clonar do repositório.
- InstanceType (String, Default: `t3.micro`)  
  - Tipo da instância EC2.
- UseLatestAmi (String: `Yes` | `No`, Default: `Yes`)  
  - Quando `Yes`, obtém a AMI Amazon Linux 2 mais recente via SSM. Se `No`, usa `ImageIdParameter`.
- LatestAmiParam (SSM parameter, Default: `/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2`)  
  - Parâmetro SSM que aponta para a AMI mais recente (usado quando `UseLatestAmi=Yes`).
- ImageIdParameter (String, Default: `ami-052064a798f08f0d3`)  
  - AMI estática (usada quando `UseLatestAmi=No`). Substitua conforme sua região.
- AppAllowedCidr (String, Default: `0.0.0.0/0`)  
  - CIDR permitido para acessar a porta da aplicação (8000). Ajuste para restringir acesso se necessário.

Condições
---------
- `IsProd` — verdadeiro quando `EnvironmentType = Prod`. Controla a criação da EC2 e alguns outputs.
- `UseSsmAmi` — verdadeiro quando `UseLatestAmi = Yes`. Controla se a AMI será buscada via SSM.
- `HasKeyName` — verdadeiro quando `KeyName` não está vazio.

Recursos criados
----------------
- AWS::EC2::SecurityGroup (JulianaAppSecurityGroup)  
  - Regras de entrada: SSH (22) para `MyIp` e App (8000) para `AppAllowedCidr`.
- AWS::EC2::Instance (EC2Instance) — criado apenas se `IsProd = true`  
  - Propriedades principais: `InstanceType`, `ImageId` (SSM ou estático), `KeyName` (opcional), `SubnetId`, `AssociatePublicIpAddress`, SecurityGroup.
  - `UserData` (base64): script que detecta `yum`/`apt`, instala Docker e Git, inicia o Docker, clona/atualiza o repositório em `/opt/juliana-app`, builda a imagem Docker e inicia o container `juliana` mapeando `8000:80`.

Outputs
-------
- `SecurityGroupId` — ID do Security Group criado (sempre).
- `InstanceId` — ID da instância EC2 (apenas quando `EnvironmentType=Prod`).
- `PublicIp` — IP público da instância EC2 (apenas quando `EnvironmentType=Prod`).
- `WebUrl` — URL pública da aplicação (http://<PublicIp>:8000) (apenas quando `EnvironmentType=Prod`).

Exemplos de uso (AWS CLI)
-------------------------

1) Validar o template localmente:
```bash
aws cloudformation validate-template --template-body file://ec2-juliana-teste.yaml
```

2) Criar/atualizar stack usando AMI via SSM (padrão — `UseLatestAmi=Yes`):
```bash
aws cloudformation deploy \
  --stack-name ec2-juliana-teste \
  --template-file ec2-juliana-teste.yaml \
  --parameter-overrides \
    EnvironmentType=Prod \
    KeyName=meu-keypair \
    VpcId=vpc-0123456789abcdef0 \
    SubnetId=subnet-0123456789abcdef0 \
    MyIp=1.2.3.4/32 \
    RepoUrl=https://github.com/SeuUsuario/seu-repo \
    Branch=main \
    AppAllowedCidr=0.0.0.0/0 \
    UseLatestAmi=Yes \
  --region us-east-1
```

3) Criar stack usando AMI estática (`UseLatestAmi=No`):
```bash
aws cloudformation deploy \
  --stack-name ec2-juliana-teste \
  --template-file ec2-juliana-teste.yaml \
  --parameter-overrides \
    EnvironmentType=Prod \
    KeyName=meu-keypair \
    VpcId=vpc-0123456789abcdef0 \
    SubnetId=subnet-0123456789abcdef0 \
    MyIp=1.2.3.4/32 \
    UseLatestAmi=No \
    ImageIdParameter=ami-0abcdef0123456789 \
  --region us-east-1
```

4) Obter outputs (IP público / URL):
```bash
aws cloudformation describe-stacks --stack-name ec2-juliana-teste --query "Stacks[0].Outputs" --output table
```

5) Deletar stack (limpeza):
```bash
aws cloudformation delete-stack --stack-name ec2-juliana-teste
```
