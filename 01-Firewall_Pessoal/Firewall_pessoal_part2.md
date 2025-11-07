Ótimo projeto! Montar um appliance de rede com o Ubuntu Server 24.04 LTS é uma escolha robusta e flexível.Abaixo, apresento a lista de programas, um esboço de como configurar e a documentação para referência futura, tudo formatado em Markdown para sua consulta.🛠️ Lista de Programas (Software Necessário)Para atender a todos os seus requisitos, você precisará dos seguintes pacotes no Ubuntu Server 24.04:FunçãoPrograma SugeridoDescriçãoFirewall de Borda (NAT, Filtros)nftables (nativo no Ubuntu 24)Substitui o iptables, mais moderno e flexível para regras de firewall e NAT (masquerading).Servidor DHCPisc-dhcp-server ou isc-kea-dhcpDistribui endereços IP para as redes LAN. O ISC DHCP é o clássico, e o Kea é o sucessor moderno e de alto desempenho.Servidor DNS (Cache/Forwarding)bind9 (ou unbound)Atuará como DNS recursivo/cache, encaminhando (forwarding) as requisições para o DNS da operadora WAN.Proxy Reversonginx ou apache2 (mod_proxy)O Nginx é amplamente preferido para Proxy Reverso devido à sua eficiência e desempenho.VPN Ponto-a-Ponto/Acesso RemotowireguardSolução VPN moderna, simples, rápida e segura. Nativa no Kernel Linux.Gerenciamento Web (DHCP/DNS/Firewall)Webmin ou CockpitPara fixar IPs no DHCP de forma fácil via interface web. O Webmin oferece módulos mais completos para gerenciar a maioria dos serviços. O Cockpit é mais moderno, nativo do Linux, mas pode precisar de complementos. Recomendo o Webmin para seu objetivo.Logs e Relatórios de AcessoSquid (para log de acesso web) + Lightsquid ou Sarg (para análise de logs)O Squid atua como proxy (não reverso) para fazer o log detalhado dos acessos de saída das redes LAN. Se você não quer forçar o proxy nos clientes, o relatório será limitado. Obrigatório para o requisito de relatório de sites visitados.⚙️ Como Configurar (Visão Geral)1. Configuração de Rede (Interfaces)Use o Netplan (/etc/netplan/*.yaml) para configurar as interfaces. O Ubuntu 24.04 utiliza o Netplan por padrão.InterfaceRedeEndereço (Exemplo)Funçãoeth0 (WAN)Dinâmica ou Estática da OperadoraIP_DA_WAN/CIDRGateway/Uplink para a Internet.eth1 (LAN1 - Wifi)10.16.100.0/2410.16.100.1/24Servidor DHCP e Gateway para a Rede Doméstica.eth2 (LAN2 - Hypervisor)172.16.254.0/24172.16.254.1/24Servidor DHCP e Gateway para a Rede Hypervisor.Habilitar o Encaminhamento de IP (IP Forwarding):É essencial para o roteamento (Firewall).Bashsudo sysctl -w net.ipv4.ip_forward=1
# Para tornar persistente, edite /etc/sysctl.conf
# Descomente ou adicione:
# net.ipv4.ip_forward = 1
2. Firewall de Borda (nftables)O nftables será o coração do seu appliance. Você precisará de:NAT (Masquerading): Permitir que as redes internas (LAN1 e LAN2) acessem a internet através do IP da WAN.Regras de Filtro: Controlar o tráfego entre WAN, LAN1 e LAN2.Regra Principal de NAT/Masquerading:Bash# Na tabela 'ip filter' ou crie uma nova 'nat'
# Supondo que 'eth0' seja sua interface WAN
sudo nft add rule ip nat postrouting oifname "eth0" masquerade
sudo systemctl enable nftables
sudo systemctl start nftables
Regras de Filtro (Exemplo - Regras de Acesso):Rede 10.16.100.0/24 (LAN1) para 172.16.254.0/24 (LAN2): Acesso irrestrito (ACCEPT).Rede 172.16.254.0/24 (LAN2) para 10.16.100.0/24 (LAN1): Acesso negado, exceto por regras específicas (DROP/REJECT por padrão, a menos que você adicione regras de ACCEPT).Restrição LAN2 para Internet: Você precisará de regras para permitir apenas portas e protocolos específicos (e.g., DNS, HTTP/S, SSH) de saída (oifname eth0).3. DHCP (isc-dhcp-server ou kea) e DNS (bind9)Instale o serviço e configure os pools de IP para cada interface LAN.Fixar Endereços IP (IP Fixo):O Webmin será seu melhor amigo aqui. No módulo DHCP Server, você pode criar facilmente entradas de host fixo (mapeamento MAC -> IP) para as máquinas que não devem pegar IP dinâmico.Configuração DNS (bind9 - Exemplo de Forwarding):Edite o arquivo de configuração do bind9 (geralmente named.conf.options) para configurar o forwarding para os DNS da sua operadora WAN.Snippet de códigooptions {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    // IPs do seu DNS da operadora WAN
    forwarders {
        IP_DNS_OPERADORA_1;
        IP_DNS_OPERADORA_2;
    };
    forward only;
};
No DHCP: Configure o DHCP para entregar o IP do próprio servidor (10.16.100.1 e 172.16.254.1) como o servidor DNS primário.4. Proxy Reverso (nginx)Instale o nginx e configure o bloco server para o domínio que você deseja resolver, encaminhando (proxy_pass) a requisição para a VM correta no seu Hypervisor (Rede LAN2).Exemplo de Configuração Nginx:Nginxserver {
    listen 80;
    server_name seu_dominio.com; # ou o IP WAN se for por IP

    location / {
        # Encaminha o tráfego para a VM01 (172.16.254.10)
        proxy_pass http://172.16.254.10:80; 
        # Configurações de cabeçalho
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
5. Relatórios de Sites Visitados (Squid e Lightsquid)Para ter um relatório de todos os sites visitados, é necessário que o tráfego web das redes LAN passe por um Proxy explícito.Instale o squid: Configure-o para escutar nas interfaces LAN1 e LAN2.Configure o Navegador/DHCP: Nas configurações de DHCP, você pode tentar configurar a opção de Proxy Automático (PAC file), ou configurar o proxy manualmente nos dispositivos.Instale lightsquid ou sarg: Essas ferramentas processam os logs detalhados do Squid para gerar relatórios visuais (em HTML) de acesso.

6. Gerenciamento Web (Webmin)O Webmin é ideal para gerenciar o DHCP e fixar IPs.Bash
# Adicionar o repositório Webmin
# Baixar a chave
sudo wget -q http://www.webmin.com/jcameron-key.asc -O- | sudo apt-key add -
# Adicionar a fonte
sudo add-apt-repository "deb [arch=amd64] http://download.webmin.com/download/repository sarge contrib"
# Instalar
sudo apt update
sudo apt install webmin
Acesse via navegador em https://[IP_DO_SEU_SERVIDOR]:10000.

7. VPN com Acesso Transparente (WireGuard)O WireGuard é nativo no Kernel do Ubuntu 24. Você vai configurar uma interface VPN virtual (ex: wg0).Geração de Chaves: Gere as chaves pública e privada para o servidor e para o cliente (seu notebook).Configuração de Servidor (/etc/wireguard/wg0.conf):Endereço: Adicione um IP para o túnel (ex: 10.0.1.1/24).Masquerading/PostUp: Adicione regras no WireGuard para que o tráfego vindo do túnel possa sair pela sua WAN (eth0) e para rotear o tráfego para as redes LAN internas (10.16.100.0/24 e 172.16.254.0/24).[Peer] Cliente: Adicione a chave pública do seu notebook.Configuração de Cliente (Seu Notebook):AllowedIPs: Defina 0.0.0.0/0 para todo o tráfego passar pela VPN, OU defina apenas as redes que você quer acessar (10.16.100.0/24, 172.16.254.0/24).Endpoint: IP_PÚBLICO_DA_WAN:PORTA_DO_WIREGUARD.Acesso Transparente (RDP - 3389):Como o VPN te dará um IP na rede 10.0.1.x, o acesso à VM 172.16.254.10 (RDP na porta 3389) será transparente se você garantir que:O servidor Ubuntu tenha a rota correta para a rede 172.16.254.0/24 (o que acontece naturalmente se for a interface eth2).O firewall (nftables) do Ubuntu permita o tráfego da rede VPN (10.0.1.x) para a rede LAN2 (172.16.254.x) na porta 3389.O firewall da própria VM 172.16.254.10 permita conexões RDP da rede VPN.

