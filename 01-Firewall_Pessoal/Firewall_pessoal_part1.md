📝 Documento Final em MarkDown para ConsultaMarkdown# 🌐 Ubuntu Server 24.04 Appliance de Rede

## 🎯 Visão Geral do Projeto

Este documento resume a configuração de um Ubuntu Server 24.04 LTS para atuar como firewall, roteador, DHCP, DNS, Proxy Reverso e Servidor VPN (WireGuard), atendendo às necessidades de duas redes internas (LAN1 - Doméstica e LAN2 - Hypervisor).

---

## ⚙️ I. Configuração de Redes

### 1. Interfaces Lógicas

| Interface | Rede | Endereço (Servidor) | Uso |
| :--- | :--- | :--- | :--- |
| **WAN (Ex: eth0)** | Operadora | DHCP / IP Estático WAN | Internet Uplink |
| **LAN1 (Ex: eth1)** | 10.16.100.0/24 | 10.16.100.1/24 | Rede Doméstica/Wifi |
| **LAN2 (Ex: eth2)** | 172.16.254.0/24 | 172.16.254.1/24 | Rede Hypervisor |
| **VPN (Ex: wg0)** | 10.0.1.0/24 | 10.0.1.1/24 | Acesso Remoto WireGuard |

### 2. Habilitar Roteamento (IP Forwarding)

```bash
# Habilitar em tempo de execução
sudo sysctl -w net.ipv4.ip_forward=1

# Tornar persistente
sudo nano /etc/sysctl.conf 
# Descomentar a linha:
# net.ipv4.ip_forward = 1
🛡️ II. Firewall de Borda (nftables)nftables é a ferramenta de firewall recomendada.1. Instalação e HabilitaçãoBashsudo apt update
sudo apt install nftables
sudo systemctl enable nftables
sudo systemctl start nftables
2. Masquerading (NAT para a Internet)Bash# Criação de uma tabela NAT e regra de Masquerading
sudo nft add table ip nat
sudo nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
# Substitua 'eth0' pela sua interface WAN
sudo nft add rule ip nat postrouting oifname "eth0" masquerade
3. Regras de Filtro (Requisitos Específicos)A tabela principal de filtro (filter) deve aplicar as políticas de acesso.OrigemDestinoServiço/PortaAçãoRequisitoLAN1 (10.16.100.0/24)LAN2 (172.16.254.0/24)QualquerACCEPTAcesso irrestrito LAN1 -> LAN2LAN2 (172.16.254.0/24)LAN1 (10.16.100.0/24)QualquerDROPLAN2 mais restritivaLAN2 (172.16.254.0/24)WAN (eth0)HTTP/S (80, 443), DNS (53)ACCEPTAcesso restrito à Internet (Exemplo)Lembre-se de adicionar as regras para permitir o tráfego DNS/DHCP para o próprio servidor.💾 III. Serviços de Rede (DHCP e DNS)1. Servidor DHCP (isc-dhcp-server)Bashsudo apt install isc-dhcp-server
# Edite /etc/default/isc-dhcp-server para definir as interfaces (eth1, eth2)
# Edite /etc/dhcp/dhcpd.conf para configurar os pools e o servidor DNS (10.16.100.1 e 172.16.254.1)

# Exemplo de Sub-rede LAN1 (em dhcpd.conf):
subnet 10.16.100.0 netmask 255.255.255.0 {
    range 10.16.100.100 10.16.100.200;
    option routers 10.16.100.1;
    option domain-name-servers 10.16.100.1; # O próprio servidor
}
Nota: Use o Webmin (módulo Servers > DHCP Server) para fixar endereços IP de forma amigável (Host Declarações).2. Servidor DNS (bind9)Bashsudo apt install bind9
# Edite /etc/bind/named.conf.options para configurar o Forwarding
options {
    forwarders {
        IP_DNS_OPERADORA_1;
        IP_DNS_OPERADORA_2;
    };
    forward only;
    // ... outras configurações ...
};
Objetivo: O servidor escutará na porta 53 das interfaces LAN, responderá a consultas locais e encaminhará as externas para o DNS da operadora WAN.🔄 IV. Proxy Reverso (nginx)Instale o NGINX e configure o host virtual para encaminhar as requisições para a VM de destino (e.g., uma máquina no 172.16.254.0/24).Bashsudo apt install nginx
sudo nano /etc/nginx/sites-available/reverso.conf

# Conteúdo de reverso.conf (Exemplo):
server {
    listen 80;
    server_name seu-dominio-reverso.com; 

    location / {
        proxy_pass [http://172.16.254.10](http://172.16.254.10); # IP da sua VM no Hypervisor
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        # ... outras configurações de Proxy Reverso ...
    }
}

sudo ln -s /etc/nginx/sites-available/reverso.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx
🔎 V. Relatório de Acesso Web (Squid e Relatório)Para ter um relatório detalhado de sites visitados, use o Squid como Proxy Explícito (não reverso).Bashsudo apt install squid lightsquid apache2 # Lightsquid requer Apache/outro webserver

# 1. Configurar o Squid:
# Edite /etc/squid/squid.conf para definir as ACLs de acesso (permitindo LAN1 e LAN2)
# Configure o Squid para logar os acessos.
# A porta padrão é 3128.

# 2. Configurar o Lightsquid:
# Configure o lightsquid para ler os logs do squid e gerar os relatórios HTML.
# Configurar o Apache para servir o Lightsquid em uma porta/url restrita.
Aviso: Os clientes de rede (LAN1 e LAN2) deverão ser configurados (manualmente ou via DHCP/GPO) para usar o servidor 10.16.100.1:3128 como proxy.🔑 VI. VPN Ponto-a-Ponto (WireGuard)1. Instalação e ChavesBashsudo apt install wireguard
umask 077 
wg genkey | tee privatekey | wg pubkey > publickey # Gerar chaves
2. Configuração do Servidor (/etc/wireguard/wg0.conf)Snippet de código[Interface]
# Endereço da VPN (lado do Servidor)
Address = 10.0.1.1/24 
ListenPort = 51820
PrivateKey = [CHAVE_PRIVADA_DO_SERVIDOR]

# Regras de NAT/Masquerading para o tráfego da VPN
# Permite que os clientes VPN acessem a Internet pela WAN (eth0) e as redes internas (eth1/eth2)
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Peer (Seu Notebook Remoto)
[Peer]
# Chave Pública do Notebook
PublicKey = [CHAVE_PÚBLICA_DO_CLIENTE]
# IP atribuído ao Notebook (10.0.1.2) - Mantenha este IP fixo para o cliente
AllowedIPs = 10.0.1.2/32
3. Acesso Transparente (RDP/3389)Com o WireGuard configurado acima, seu notebook (IP 10.0.1.2) pode acessar a VM 172.16.254.10 na porta 3389 de forma transparente, desde que você crie uma regra de firewall para permitir o tráfego da rede VPN para a LAN2.Regra nftables (Exemplo):Bash# Permitir conexões RDP da rede VPN (10.0.1.0/24) para LAN2 (172.16.254.0/24)
sudo nft add rule ip filter forward ip saddr 10.0.1.0/24 ip daddr 172.16.254.0/24 tcp dport 3389 counter accept
🖥️ VII. Gerenciamento Web (Webmin)O Webmin facilita a configuração do DHCP (fixar IPs), DNS e, em menor grau, o Firewall (via módulo Netfilter/iptables Legacy, mas nftables é preferível por CLI/Arquivo).Bash# Siga a documentação para adicionar o repositório e instalar:
sudo apt install webmin

# Acesso: https://[IP_DO_SEU_SERVIDOR]:10000
Que outras funções de rede você gostaria de restringir ou permitir no seu firewall nftables? Posso gerar um template de configuração inicial para você.