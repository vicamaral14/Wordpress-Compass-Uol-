# WordPress em Alta Disponibilidade na AWS

**Programa de Bolsas - DevSecOps**  
---

## 1. Objetivo do Projeto
Este projeto tem como objetivo implantar **WordPress** na AWS com **alta disponibilidade**, **escalabilidade automática** e **tolerância a falhas**, utilizando serviços gerenciados.  

A arquitetura garante que a aplicação continue disponível mesmo em caso de falhas em instâncias ou zonas de disponibilidade.

---

## 2. Arquitetura do Projeto
A arquitetura inclui:  

- **EC2 + Auto Scaling Group (ASG):** instâncias WordPress que escalam conforme a demanda.  
- **Application Load Balancer (ALB):** distribui tráfego e realiza health checks.  
- **Amazon RDS Multi-AZ:** banco MySQL/MariaDB altamente disponível.  
- **Amazon EFS:** sistema de arquivos compartilhado para uploads do WordPress.  
- **VPC Personalizada:** subnets públicas para ALB, privadas para EC2 e RDS, IGW e NAT Gateway.  

### 2.1 Componentes AWS

| Serviço | Função | 
|---------|-------|
| **VPC** | Rede isolada com 2 AZs, 4 subnets, IGW e NAT Gateway | 
| **RDS** | Banco de dados MySQL/MariaDB Multi-AZ | 
| **EFS** | Sistema de arquivos NFS compartilhado | 
| **EC2 + ASG** | Instâncias WordPress escaláveis automaticamente | 
| **ALB** | Balanceador de carga para distribuir tráfego |

---

## 3. Passo a Passo do Projeto

### 3.1 Teste Local
1. Baixar WordPress:
```bash
docker pull wordpress
