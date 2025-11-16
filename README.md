# 🚀 Laboratório AWS: EC2, EBS e Snapshots com CloudFormation

Este projeto automatiza o provisionamento de um servidor web simples na AWS, conforme descrito em um exercício de laboratório do curso da EDN para a trilha de Arquiteto de Soluções.
O objetivo é criar uma instância EC2, anexar um volume EBS adicional para dados e configurar backups automáticos.

Todo o provisionamento é feito usando **Infraestrutura como Código (IaC)** com um único template do AWS CloudFormation (`lab-ec2-ebs.yaml`) e a AWS CLI.

[![IaC: CloudFormation](https://img.shields.io/badge/IaC-CloudFormation-FF9900?logo=amazon-aws)](https://aws.amazon.com/cloudformation/)

---

## 📋 O que este template cria?

O arquivo `lab-ec2-ebs.yaml` provisiona os seguintes recursos:

* **Instância EC2:** Uma instância `t2.micro` (ou `t3.micro`) usando a AMI do Amazon Linux 2.
* **Volume EBS:** Um volume de dados `gp2` de 8 GB separado.
* **Anexação de Volume:** Um recurso `VolumeAttachment` para "ligar" o volume à instância no dispositivo `/dev/sdf`.
* **Security Group:** Um grupo de segurança que permite acesso SSH (porta 22) a partir de qualquer IP (0.0.0.0/0).
* **Script UserData:** Um script de inicialização que, automaticamente:
    1.  Formata o novo volume com o sistema de arquivos `ext4`
    2.  Cria o diretório `/mnt/dados`.
    3.  Monta o volume no diretório `/mnt/dados`.
    4.  Atualiza o `/etc/fstab` para remontar o volume automaticamente após reinicializações.
    5.  Instala um servidor web Apache (`httpd`) para simular o cenário.
* **IAM Role:** Uma role para o AWS Data Lifecycle Manager (DLM).
  **Política de Snapshot (DLM):** Uma política que cria snapshots diários automaticamente  dos volumes marcados com a tag `BackupPolicy: Daily`.
<img width="1911" height="848" alt="Captura de tela 2025-11-16 192445" src="https://github.com/user-attachments/assets/69c636d0-000d-41dc-8b65-8cba81225f30" />
---

## 🛠️ Como Executar

### Pré-requisitos

* Uma conta da AWS.
* (https://aws.amazon.com/cli/) instalada e configurada com suas credenciais.

### Passo 1: Criar o Par de Chaves (Key Pair)

Este é o único passo manual. Precisamos criar um par de chaves para acessar a instância. O arquivo `.pem` será salvo localmente.

```bash
# Define o nome da chave
KEY_NAME="lab-ec2-key"

# Cria o par de chaves na AWS e salva o arquivo .pem
aws ec2 create-key-pair \
    --key-name $KEY_NAME \
    --query 'KeyMaterial' \
    --output text > ${KEY_NAME}.pem

# Ajusta as permissões do arquivo da chave (Obrigatório)
chmod 400 ${KEY_NAME}.pem
```
Passo 2: Implantar a Stack do CloudFormation
Este comando lê o arquivo YAML e cria todos os recursos na AWS.

```Bash

aws cloudformation create-stack \
    --stack-name meu-lab-ec2 \
    --template-body file://lab-ec2-ebs.yaml \
    --parameters ParameterKey=KeyName,ParameterValue=$KEY_NAME \
    --capabilities CAPABILITY_IAM
(A flag --capabilities CAPABILITY_IAM é necessária porque estamos criando uma Role).
```

Passo 3: Aguardar a Criação
Aguarde a stack ser criada. Isso pode levar alguns minutos.

```Bash

aws cloudformation wait stack-create-complete --stack-name meu-lab-ec2
echo "✅ Stack criada com sucesso!"
```

🔎 Verificação
Após a criação, vamos conectar na instância e verificar se o volume foi montado.
<img width="1425" height="440" alt="Captura de tela 2025-11-16 192911" src="https://github.com/user-attachments/assets/a0410fd4-3c79-4f41-9858-635f6d5cb051" />

Obtenha o IP Público da Instância:

```Bash

PUBLIC_IP=$(aws cloudformation describe-stacks --stack-name meu-lab-ec2 --query "Stacks[0].Outputs[?OutputKey=='PublicIP'].OutputValue" --output text)
echo "Conecte em: $PUBLIC_IP"
```
Conecte-se via SSH:


```Bash

ssh -i "${KEY_NAME}.pem" ec2-user@$PUBLIC_IP

```
Verifique os Volumes (Dentro da Instância): Uma vez conectado, rode lsblk. A saída deve mostrar o volume xvdf montado em /mnt/dados.

```Bash

[ec2-user@...]$ lsblk
NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda    202:0    0   8G  0 disk
└─xvda1 202:1    0   8G  0 part /
xvdf    202:80   0   8G  0 disk /mnt/dados  <-- SUCESSO!
🎯 Tarefa do Laboratório: Snapshot Manual
O laboratório também pedia para criar um snapshot manual.
```
(No seu terminal local) Obtenha o ID do volume:

```Bash

VOLUME_ID=$(aws ec2 describe-volumes --filters "Name=tag:Name,Values=lab-data-volume" --query "Volumes[0].VolumeId" --output text)
```
Crie o snapshot com a descrição solicitada (sem acentos):

```Bash

aws ec2 create-snapshot \
    --volume-id $VOLUME_ID \
    --description "Nao apague, risco de demissao" \
    --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=Snapshot-Manual-Lab},{Key=FinOps,Value=Lab-EC2}]'
```

🚨 ALERTA DE SEGURANÇA: .gitignore
Este repositório contém um arquivo .gitignore que está configurado para IGNORAR todos os arquivos que terminam com *.pem.

NUNCA, EM HIPÓTESE ALGUMA, remova esta linha ou envie seu arquivo .pem para o GitHub.

Vazar uma chave privada é o mesmo que vazar a senha mestra do seu servidor. Bots maliciosos escaneiam o GitHub constantemente em busca dessas chaves.

🧹 Limpeza
Para evitar cobranças, delete TODOS os recursos criados por este lab.

Delete a Stack do CloudFormation: Este comando deleta a Instância EC2, o Volume EBS, a Role, a Política DLM e o Security Group.

```Bash

aws cloudformation delete-stack --stack-name meu-lab-ec2

# Aguarde a exclusão
aws cloudformation wait stack-delete-complete --stack-name meu-lab-ec2
echo "Stack deletada."
```
<img width="1905" height="847" alt="Captura de tela 2025-11-16 201217" src="https://github.com/user-attachments/assets/1132c2c5-b8f0-43ce-98a8-d968640c3830" />

Delete o Par de Chaves: A stack não deleta o par de chaves.

```Bash

aws ec2 delete-key-pair --key-name $KEY_NAME
Delete o Snapshot Manual: Snapshots também são excluídos manualmente.
```
```Bash

# Encontre o ID do snapshot
SNAPSHOT_ID=$(aws ec2 describe-snapshots --filters "Name=description,Values=Nao apague, risco de demissao" --query "Snapshots[0].SnapshotId" --output text)

# Delete o snapshot
aws ec2 delete-snapshot --snapshot-id $SNAPSHOT_ID
echo "Snapshot manual deletado."
```
<img width="1041" height="152" alt="Captura de tela 2025-11-16 201726" src="https://github.com/user-attachments/assets/b7397728-bace-47ed-9fe6-4a5b13308717" />
<img width="1912" height="234" alt="Captura de tela 2025-11-16 201944" src="https://github.com/user-attachments/assets/109e4408-948c-48d7-9813-b58dc2f43efc" />
