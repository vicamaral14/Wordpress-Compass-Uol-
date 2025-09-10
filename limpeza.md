## Limpeza (ordem recomendada)
1. Terminate ASG (delete group) — certifique que instâncias terminem
2. Delete ALB
3. Delete Target Group
4. Delete Launch Template
5. Delete EFS (remova mount targets primeiro, ou EFS removerá automaticamente)
6. Delete RDS (verifique snapshots/backups antes de deletar)
7. Terminate bastion instance
8. Delete Security Groups
9. Delete NAT Gateway
10. Release Elastic IP
11. Delete Route Tables (rt-public, rt-private)
12. Detach & delete Internet Gateway
13. Delete Subnets
14. Delete VPC
