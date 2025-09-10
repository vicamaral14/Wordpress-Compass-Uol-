# WordPress em Alta Disponibilidade na AWS

**Programa de Bolsas - DevSecOps**  

Bolsista: Victória Do Amaral

Este projeto implanta WordPress na AWS de forma escalável, tolerante a falhas e de alta disponibilidade.  

Para detalhes completos, veja os arquivos:  
- [Componentes AWS](01_componentes.md)  
- [Passo a Passo](03_passos_projeto.md)  
- [User Data Script](03_user_data.md)
- [Limpeza recomendada](limpeza.md)

## Notas importantes e dicas
- **Testes:** Antes de avançar para cada etapa, faça testes com o que vc criou.
- **Tempo:** NAT Gateway, EFS mount targets e RDS podem levar alguns minutos para ficarem prontos. Planeje esperar alguns minutos entre passos.
- **Custos:** NAT Gateway, EFS, ALB e RDS custam em produção — desligue/limpe recursos quando testar para evitar cobranças.
- **Segurança:** restrinja o acesso SSH do bastion ao seu IP; permita tráfego ao RDS apenas a partir do SG das instâncias.
- **Substitua AMIs e IDs:** AMI do Ubuntu e IDs do EFS/DB são regionais — substitua no user-data e comandos.

