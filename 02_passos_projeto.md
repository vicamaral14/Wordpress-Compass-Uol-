## 1) Criar VPC
**Console path:** VPC → Your VPCs → Create VPC
- Name tag: `wp-vpc`
- IPv4 CIDR block: `10.0.0.0/16`
- Tenancy: Default
- Após criar: habilite **DNS hostnames** e **DNS resolution** (Actions → Edit DNS hostnames / Edit DNS resolution) — importante para ALB, EFS e comunicação.

---

## 2) Criar sub-redes
**Console path:** VPC → Subnets → Create subnet
Crie 4 sub-redes no VPC `wp-vpc`:
- `Public-1a` (AZ = 1a) — `10.0.1.0/24` — **Auto-assign Public IPv4 address: ENABLED**
- `Public-1b` (AZ = 1b) — `10.0.2.0/24` — **Auto-assign Public IPv4 address: ENABLED**
- `Private-1a` (AZ = 1a) — `10.0.101.0/24` — **Auto-assign: DISABLED**
- `Private-1b` (AZ = 1b) — `10.0.102.0/24` — **Auto-assign: DISABLED**

---

## 3) Internet Gateway
**Console path:** VPC → Internet Gateways
- Create internet gateway (nome: `igw-wp`) → Attach to `wp-vpc`.

---

## 4) Tabelas de rotas
**Console path:** VPC → Route Tables

### Tabela pública
- Create route table → Name: `rt-public` → VPC: `wp-vpc`.
- Edit routes: adicionar `0.0.0.0/0` → Target: `igw-wp`.
- Associate → associe `rt-public` às sub-redes `Public-1a` e `Public-1b`.

### Tabela privada
- Create route table → Name: `rt-private` → VPC: `wp-vpc`.
- **Não** adicione ainda a rota 0.0.0.0/0 — aguarde a criação do NAT Gateway.
- Depois de criar o NAT, adicione rota `0.0.0.0/0` → Target: **NAT Gateway** (selecione o NAT criado).
- Associate → associe `rt-private` às sub-redes `Private-1a` e `Private-1b`.

> Observação: se você quiser sub-redes privadas com acesso à Internet somente via NAT, mantenha `Public-1x` com a associação ao `rt-public` e `Private-1x` ao `rt-private`.

---

## 5) Elastic IP + NAT Gateway
**Console path:** EC2 → Network & Security → Elastic IPs
- Allocate Elastic IP — reserve para o NAT.

**Console path:** VPC → NAT Gateways (ou EC2 → NAT Gateways)
- Create NAT Gateway
  - Subnet: `Public-1a`
  - Elastic IP: selecione o EIP alocado
- Aguarde o estado `Available`.
- Atualize a rota `rt-private` adicionando `0.0.0.0/0` → NAT Gateway.

---

## 6) Security Groups (SG)
**Console path:** EC2 → Network & Security → Security Groups
Crie os SGs abaixo no VPC `wp-vpc`.

**sg-bastion** (para o host bastion)
- Inbound:
  - SSH (TCP 22) — Source: **seu IP** (ex.: `203.0.113.1/32`) — restrinja ao máximo
- Outbound: All traffic — `0.0.0.0/0`

**sg-alb** (para o ALB)
- Inbound:
  - HTTP (TCP 80) — Source: `0.0.0.0/0` (se público)
  - (opcional) HTTPS (TCP 443) — Source: `0.0.0.0/0`
- Outbound: All traffic

**sg-ec2** (para instâncias que rodarão WordPress/app)
- Inbound:
  - HTTP (TCP 80) — Source: **sg-alb** (referência de security group)
  - SSH (TCP 22) — Source: **sg-bastion** (permitir ssh apenas do bastion)
- Outbound: All

**sg-rds** (para banco de dados)
- Inbound:
  - MySQL/Aurora (TCP 3306) — Source: **sg-ec2**
- Outbound: All

**sg-efs** (para EFS mount targets)
- Inbound:
  - NFS (TCP 2049) — Source: **sg-ec2**
- Outbound: All

> Dica: ao adicionar regras que usam outro SG como origem, selecione o ID do SG (ex.: `sg-0abc1234`) no console.

---

## 7) Bastion Host (EC2)
**Console path:** EC2 → Instances → Launch Instances
- AMI: Ubuntu 24.04 LTS (escolha AMI correta por região)
- Instance type: `t3.micro` ou similar (teste)
- Network: `wp-vpc` → Subnet: `Public-1a`
- Auto-assign Public IP: **Enable**
- Key pair: `ec2-wordpress`
- Security group: `sg-bastion`
- Name: `bastion-wp`

**Conectar via SSH (do seu PC):**
```bash
chmod 400 ec2-wordpress.pem
ssh -i ec2-wordpress.pem ubuntu@<BASTION_PUBLIC_IP>
```

> Use o bastion para acessar instâncias nas sub-redes privadas (via `ssh -A` ou `ProxyJump`).

---

## 8) RDS — MySQL (instância na sub-rede privada)
**Console path:** RDS → Databases → Create database
- Engine: **MySQL**
- Creation method: Standard create
- DB instance identifier: `wp-db`
- Credentials: crie usuário e senha (anote com segurança)
- DB instance class: `db.t3.medium` (ajuste conforme necessidade)
- Multi-AZ: **Opcional** (recomendado para produção)
- Storage: gp3 por exemplo
- Connectivity:
  - VPC: `wp-vpc`
  - DB subnet group: crie um DB subnet group (RDS → Subnet groups) com as sub-redes `Private-1a` e `Private-1b` antes de criar o DB
  - Public accessibility: **No**
  - VPC security groups: `sg-rds`
- Additional configuration: nome inicial do banco (ex.: `wordpress`)

> Observação: ative backups automáticos conforme política da sua empresa. Verifique se a **DB subnet group** usa apenas as sub-redes privadas.

---

## 9) EFS (regional) e mount targets
**Console path:** EFS → File systems → Create file system
- Escolha VPC `wp-vpc`
- Selecione as sub-redes (crie mount targets) — marque `Private-1a` e `Private-1b`
- Security group: `sg-efs` (o console pedirá qual SG aplicar aos mount targets)
- Create

> O EFS cria interfaces de rede (ENIs) nas sub-redes selecionadas — confirme que há ENIs nas sub-redes privadas.

### Montagem no EC2 (exemplo rápido)
No `wp-template` (user-data) você montará o EFS. Exemplo de comando para testar manualmente (substitua `fs-XXXXXXX`):
```bash
sudo apt update
sudo apt install -y amazon-efs-utils nfs-common
sudo mkdir -p /var/www/html
sudo mount -t efs fs-XXXXXXX:/ /var/www/html
# ou via DNS
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 fs-XXXXXXX.efs.<region>.amazonaws.com:/ /var/www/html
```

---

## 10) Launch Template (`wp-template`)
**Console path:** EC2 → Launch Templates → Create launch template
- Name: `wp-template`
- AMI: Ubuntu 24.04 LTS (escolha a AMI correta para a sua região)
- Instance type: (ex.: t3.micro / t3.small conforme teste)
- Key pair: `ec2-wordpress`
- Security group: sugerido `sg-ec2` (ou deixe vazio e associe no ASG)
- User data: cole o seu script de inicialização (User Data) — script **bash** que instala nginx/apache, mount do EFS, configura PHP/WordPress, registra tags, etc.

---

## 11) Target Group (`wp-target-group`)
**Console path:** EC2 → Load Balancing → Target Groups → Create target group
- Name: `wp-target-group`
- Target type: **Instance**
- Protocol: HTTP, Port: `80`
- VPC: `wp-vpc`
- Health checks:
  - Protocol: HTTP
  - Path: `/`
  - Success codes: `200-399`
- Create

---

## 12) Application Load Balancer (ALB)
**Console path:** EC2 → Load Balancing → Load Balancers → Create Load Balancer → Application Load Balancer
- Name: `wp-alb`
- Scheme: Internet-facing
- IP address type: IPv4
- Listeners: HTTP :80 (pode adicionar HTTPS :443 depois)
- Availability Zones: selecione as sub-redes públicas `Public-1a` e `Public-1b`
- Security groups: `sg-alb`
- Configure routing: aponte o listener HTTP:80 para o target group `wp-target-group` (ou deixe para depois e configure listener)
- Create

> Anote o DNS name do ALB — será o front-end público do site.

---

## 13) Auto Scaling Group (ASG) — `wp-asg`
**Console path:** EC2 → Auto Scaling → Create Auto Scaling group
- Name: `wp-asg`
- Launch template: selecione `wp-template` (escolha versão que contém user-data)
- VPC: `wp-vpc`
- Subnets: escolha `Private-1a` e `Private-1b`
- Load Balancer: selecione o ALB e associe `wp-target-group`
- Group size: Desired = `1`, Min = `1`, Max = `2` (ajuste conforme necessidade)
- Configure scaling policies: opcional — pode criar política baseada em CPU ou em ALB request count
- Create

> O ASG criará instâncias nas sub-redes privadas; o ALB (em sub-redes públicas) encaminhará tráfego para as instâncias.

---
## 14) Testes e validação
- **ALB:** pegue o DNS do ALB (`wp-alb-xxxx.elb.amazonaws.com`) e abra no navegador. Deve responder com página criada pelo User Data (ex.: nginx) ou WordPress se o script instalou.
- **Target Group:** verifique se instâncias aparecem como **healthy** (Targets → Health checks). Se unhealthy, verifique o security group (sg-ec2) e se a aplicação está ouvindo na porta 80.
- **RDS:** a partir de uma instância privada, teste conexão `mysql -h <RDS_ENDPOINT> -u <user> -p`.
- **EFS:** na instância, verifique `df -h` e `ls /var/www/html`.
