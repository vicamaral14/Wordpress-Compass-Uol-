Passo 1: Preparando o Ambiente Local com Docker
Antes de subir tudo para a nuvem, é uma ótima prática testar a aplicação localmente. O Docker nos permite fazer isso de forma rápida e isolada.

Baixe a imagem do WordPress:

Bash

docker pull wordpress
Este comando baixa a imagem oficial do WordPress para o seu computador. Pense nisso como baixar um arquivo de instalação pré-configurado.

Inicie os serviços do Docker:

Bash

docker-compose up -d
Este comando usa um arquivo de configuração (chamado docker-compose.yml) para iniciar o WordPress e o banco de dados (MySQL/MariaDB) de uma só vez, em segundo plano (-d).

Passo 2: Criando a Estrutura de Rede na AWS (VPC)
A Virtual Private Cloud (VPC) é a sua "rede privada" na nuvem. Nela, vamos isolar nossos recursos e controlar o tráfego.

Crie sua VPC: Vá até o serviço de VPC na AWS e crie uma nova. Pense nela como a base de toda a sua infraestrutura.

Crie 2 Zonas de Disponibilidade (AZs): As AZs são locais físicos separados. Usar duas garante que, se uma falhar, a outra continuará funcionando, aumentando a disponibilidade.

Crie 4 Sub-redes: Sub-redes são divisões da sua VPC.

2 sub-redes públicas: Recursos que precisam de acesso direto à internet (como o Load Balancer) serão colocados aqui.

2 sub-redes privadas: Recursos que não precisam de acesso direto à internet (como as instâncias do WordPress e o banco de dados) serão colocados aqui, por segurança.

Configure o Gateway de Internet (Internet Gateway - IGW): Conecte-o à sua VPC. Ele permite que o tráfego de internet entre e saia das suas sub-redes públicas.

Configure o Gateway NAT (NAT Gateway): Coloque-o em uma sub-rede pública. O NAT Gateway permite que os recursos das sub-redes privadas se conectem à internet (para fazer atualizações, por exemplo), mas impede que o tráfego de internet chegue até eles diretamente.

Passo 3: Criando os Serviços Principais
Agora que a rede está pronta, vamos criar os serviços essenciais para o WordPress.

Crie o Banco de Dados (RDS):

Vá para o serviço RDS e crie uma nova instância de banco de dados (MySQL ou MariaDB).

Crie um Security Group específico para o RDS. O Security Group funciona como um firewall e deve ser configurado para permitir conexões apenas a partir das instâncias EC2, ou seja, as máquinas onde o WordPress irá rodar.

Crie o Sistema de Arquivos (EFS):

O Elastic File System (EFS) permite que várias instâncias EC2 compartilhem a mesma pasta de arquivos. Isso é crucial para o WordPress, já que as imagens e plugins precisam estar acessíveis para todas as instâncias que rodam o site.

Crie o EFS e ajuste as permissões de seu Security Group para permitir acesso das instâncias EC2.

Passo 4: Criando uma Imagem Personalizada (AMI) e um Template de Lançamento
Em vez de configurar cada máquina EC2 manualmente, vamos criar um "modelo" com tudo pronto.

Crie uma AMI base (opcional, mas recomendado):

Inicie uma instância EC2 com um sistema operacional (como o Linux 2 da Amazon).

Instale o Apache, PHP e o WordPress.

Configure a instância para montar o EFS e conectar-se ao RDS.

Após a configuração, crie uma AMI (Amazon Machine Image) a partir dessa instância. Essa AMI será a sua imagem personalizada, um clone perfeito da sua máquina configurada.

Crie um Template de Lançamento (Launch Template):

Este template vai usar a sua AMI personalizada como base.

Adicione um User Data ao template. O User Data é um script que roda quando a instância é iniciada. Use-o para montar o EFS e conectar a instância ao banco de dados RDS de forma automatizada.

Salve o template com essas configurações.

Passo 5: Configurando a Escalabilidade Automática (Auto Scaling Group)
O Auto Scaling Group (ASG) é o serviço que garante que você sempre tenha o número certo de instâncias para suportar o tráfego do seu site.

Crie o Auto Scaling Group:

Associe o ASG às suas sub-redes privadas e ao seu Application Load Balancer (ALB), que criaremos a seguir.

Configure a política de escalonamento: por exemplo, aumente o número de instâncias se a utilização da CPU ultrapassar 70% e diminua quando a utilização cair.

Passo 6: Distribuindo o Tráfego com o Load Balancer
O Application Load Balancer (ALB) distribui o tráfego de internet entre as suas instâncias EC2.

Crie um Application Load Balancer:

Associe-o às suas sub-redes públicas.

Configure os Listeners: eles escutam o tráfego nas portas HTTP (80) e HTTPS (443).

Configure o ALB para encaminhar todo o tráfego para o seu Auto Scaling Group.
