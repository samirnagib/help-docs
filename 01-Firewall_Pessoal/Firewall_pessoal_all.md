Ótimo projeto\! Montar um *appliance* de rede com o Ubuntu Server 24.04 LTS é uma escolha robusta e flexível.

Abaixo, apresento a lista de programas, um esboço de como configurar e a documentação para referência futura, tudo formatado em Markdown para sua consulta.

-----

## 🛠️ Lista de Programas (Software Necessário)

Para atender a todos os seus requisitos, você precisará dos seguintes pacotes no Ubuntu Server 24.04:

| Função | Programa Sugerido | Descrição |
| :--- | :--- | :--- |
| **Firewall de Borda (NAT, Filtros)** | **`nftables`** (nativo no Ubuntu 24) | Substitui o `iptables`, mais moderno e flexível para regras de firewall e NAT (masquerading). |
| **Servidor DHCP** | **`isc-dhcp-server`** ou **`isc-kea-dhcp`** | Distribui endereços IP para as redes LAN. O ISC DHCP é o clássico, e o Kea é o sucessor moderno e de alto desempenho. |
| **Servidor DNS (Cache/Forwarding)** | **`bind9`** (ou `unbound`) | Atuará como DNS recursivo/cache, encaminhando (forwarding) as requisições para o DNS da operadora WAN. |
| **Proxy Reverso** | **`nginx`** ou **`apache2`** (`mod_proxy`) | O Nginx é amplamente preferido para Proxy Reverso devido à sua eficiência e desempenho. |
| **VPN Ponto-a-Ponto/Acesso Remoto** | **`wireguard`** | Solução VPN moderna, simples, rápida e segura. Nativa no Kernel Linux. |
| **Gerenciamento Web (DHCP/DNS/Firewall)** | **`Webmin`** ou **`Cockpit`** | Para fixar IPs no DHCP de forma fácil via interface web. O Webmin oferece módulos mais completos para gerenciar a maioria dos serviços. O Cockpit é mais moderno, nativo do Linux, mas pode precisar de complementos. **Recomendo o Webmin para seu objetivo.** |
| **Logs e Relatórios de Acesso** | **`Squid`** (para log de acesso web) + **`Lightsquid`** ou **`Sarg`** (para análise de logs) | O Squid atua como proxy (não reverso) para fazer o log detalhado dos acessos de saída das redes LAN. Se você não quer forçar o proxy nos clientes, o relatório será limitado. **Obrigatório para o requisito de relatório de sites visitados.** |

-----

## ⚙️ Como Configurar (Visão Geral)

### 1\. Configuração de Rede (Interfaces)

Use o **Netplan** (`/etc/netplan/*.yaml`) para configurar as interfaces. O Ubuntu 24.04 utiliza o Netplan por padrão.

| Interface | Rede | Endereço (Exemplo) | Função |
| :--- | :--- | :--- | :--- |
| **`eth0` (WAN)** | Dinâmica ou Estática da Operadora | IP\_DA\_WAN/CIDR | Gateway/Uplink para a Internet. |
| **`eth1` (LAN1 - Wifi)**| 10.16.100.0/24 | 10.16.100.1/24 | Servidor DHCP e Gateway para a Rede Doméstica. |
| **`eth2` (LAN2 - Hypervisor)**| 172.16.254.0/24 | 172.16.254.1/24 | Servidor DHCP e Gateway para a Rede Hypervisor. |

**Habilitar o Encaminhamento de IP (IP Forwarding):**
É essencial para o roteamento (Firewall).

```bash
sudo sysctl -w net.ipv4.ip_forward=1
# Para tornar persistente, edite /etc/sysctl.conf
# Descomente ou adicione:
# net.ipv4.ip_forward = 1
```

### 2\. Firewall de Borda (`nftables`)

O `nftables` será o coração do seu *appliance*. Você precisará de:

  * **NAT (Masquerading):** Permitir que as redes internas (LAN1 e LAN2) acessem a internet através do IP da WAN.
  * **Regras de Filtro:** Controlar o tráfego entre WAN, LAN1 e LAN2.

**Regra Principal de NAT/Masquerading:**

```bash
# Na tabela 'ip filter' ou crie uma nova 'nat'
# Supondo que 'eth0' seja sua interface WAN
sudo nft add rule ip nat postrouting oifname "eth0" masquerade
sudo systemctl enable nftables
sudo systemctl start nftables
```

**Regras de Filtro (Exemplo - Regras de Acesso):**

  * **Rede 10.16.100.0/24 (LAN1) para 172.16.254.0/24 (LAN2):** Acesso irrestrito (ACCEPT).
  * **Rede 172.16.254.0/24 (LAN2) para 10.16.100.0/24 (LAN1):** Acesso negado, exceto por regras específicas (DROP/REJECT por padrão, a menos que você adicione regras de ACCEPT).
  * **Restrição LAN2 para Internet:** Você precisará de regras para permitir apenas portas e protocolos específicos (e.g., DNS, HTTP/S, SSH) de saída (`oifname eth0`).

### 3\. DHCP (`isc-dhcp-server` ou `kea`) e DNS (`bind9`)

Instale o serviço e configure os *pools* de IP para cada interface LAN.

```
    apt install isc-dhcp-server
```


**Fixar Endereços IP (IP Fixo):**
O **Webmin** será seu melhor amigo aqui. No módulo DHCP Server, você pode criar facilmente entradas de host fixo (mapeamento MAC -\> IP) para as máquinas que não devem pegar IP dinâmico.

**Configuração DNS (`bind9` - Exemplo de Forwarding):**
Edite o arquivo de configuração do `bind9` (geralmente `named.conf.options`) para configurar o *forwarding* para os DNS da sua operadora WAN.

```conf
options {
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
```

**No DHCP:** Configure o DHCP para entregar o IP do próprio servidor (10.16.100.1 e 172.16.254.1) como o servidor DNS primário.

### 4\. Proxy Reverso (`nginx`)

Instale o `nginx` e configure o bloco `server` para o domínio que você deseja resolver, encaminhando (proxy\_pass) a requisição para a VM correta no seu Hypervisor (Rede LAN2).

**Exemplo de Configuração Nginx:**

```nginx
server {
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
```

### 5\. Relatórios de Sites Visitados (`Squid` e `Lightsquid`)

Para ter um relatório de todos os sites visitados, é necessário que o tráfego web das redes LAN passe por um Proxy explícito.

1.  **Instale o `squid`**: Configure-o para escutar nas interfaces LAN1 e LAN2.
2.  **Configure o Navegador/DHCP:** Nas configurações de DHCP, você pode tentar configurar a opção de **Proxy Automático (PAC file)**, ou configurar o proxy manualmente nos dispositivos.
3.  **Instale `lightsquid` ou `sarg`:** Essas ferramentas processam os logs detalhados do Squid para gerar relatórios visuais (em HTML) de acesso.

### 6\. Gerenciamento Web (`Webmin`)

O Webmin é ideal para gerenciar o DHCP e fixar IPs.

```bash
# Adicionar o repositório Webmin
# Baixar a chave
sudo wget -q http://www.webmin.com/jcameron-key.asc -O- | sudo apt-key add -
# Adicionar a fonte
sudo add-apt-repository "deb [arch=amd64] http://download.webmin.com/download/repository sarge contrib"
# Instalar
sudo apt update
sudo apt install webmin
```

Acesse via navegador em `https://[IP_DO_SEU_SERVIDOR]:10000`.

### 7\. VPN com Acesso Transparente (`WireGuard`)

O WireGuard é nativo no Kernel do Ubuntu 24. Você vai configurar uma interface VPN virtual (ex: `wg0`).

1.  **Geração de Chaves:** Gere as chaves pública e privada para o servidor e para o cliente (seu notebook).
2.  **Configuração de Servidor (`/etc/wireguard/wg0.conf`):**
      * **Endereço:** Adicione um IP para o túnel (ex: `10.0.1.1/24`).
      * **Masquerading/PostUp:** Adicione regras no WireGuard para que o tráfego vindo do túnel possa sair pela sua WAN (`eth0`) e para rotear o tráfego para as redes LAN internas (10.16.100.0/24 e 172.16.254.0/24).
      * **[Peer] Cliente:** Adicione a chave pública do seu notebook.
3.  **Configuração de Cliente (Seu Notebook):**
      * **AllowedIPs:** Defina `0.0.0.0/0` para todo o tráfego passar pela VPN, **OU** defina apenas as redes que você quer acessar (`10.16.100.0/24, 172.16.254.0/24`).
      * **Endpoint:** IP\_PÚBLICO\_DA\_WAN:PORTA\_DO\_WIREGUARD.

**Acesso Transparente (RDP - 3389):**
Como o VPN te dará um IP na rede `10.0.1.x`, o acesso à VM `172.16.254.10` (RDP na porta 3389) será transparente se você garantir que:

  * O servidor Ubuntu tenha a rota correta para a rede `172.16.254.0/24` (o que acontece naturalmente se for a interface `eth2`).
  * O firewall (`nftables`) do Ubuntu permita o tráfego da rede VPN (`10.0.1.x`) para a rede LAN2 (`172.16.254.x`) na porta 3389.
  * O firewall da própria VM `172.16.254.10` permita conexões RDP da rede VPN.

-----

## 🔗 Documentação para Referência Futura

| Tópico | Recurso Sugerido |
| :--- | :--- |
| **Instalação Ubuntu Server 24.04** | [Documentação Oficial do Ubuntu Server (em inglês)](https://ubuntu.com/server/docs) |
| **Configuração de Rede (Netplan)** | [Documentação Netplan (em inglês)](https://netplan.io/examples) |
| **Firewall (nftables)** | [Wiki Oficial do Ubuntu sobre nftables (em inglês)](https://www.google.com/search?q=https://help.ubuntu.com/community/Netfilter) ou [Documentação nftables.org](https://www.google.com/search?q=https://wiki.nftables.org/) |
| **DHCP (ISC DHCPD)** | [Manual do ISC DHCP Server (em inglês)](https://www.google.com/search?q=https://www.isc.org/docs/isc-dhcp-4.4.3-manual.pdf) |
| **DNS (BIND9)** | [Guia de Configuração do BIND 9 (em inglês)](https://www.google.com/search?q=https://www.isc.org/bind/documentation/) |
| **Proxy Reverso (NGINX)** | [Documentação NGINX sobre Reverse Proxy (em inglês)](http://nginx.org/en/docs/http/ngx_http_proxy_module.html) |
| **VPN (WireGuard)** | [WireGuard Quick Start (em inglês)](https://www.wireguard.com/quickstart/) |
| **Gerenciamento Web (Webmin)** | [Documentação do Webmin (em inglês)](http://www.webmin.com/docs.html) |
| **Relatórios Web (Squid)** | [Wiki do Squid Cache (em inglês)](https://www.google.com/search?q=http://www.squid-cache.org/Versions/v6/Manual/) |

-----

## 📝 Documento Final em MarkDown para Consulta

````markdown
# 🌐 Ubuntu Server 24.04 Appliance de Rede

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
````

-----

## 🛡️ II. Firewall de Borda (`nftables`)

`nftables` é a ferramenta de firewall recomendada.

### 1\. Instalação e Habilitação

```bash
sudo apt update
sudo apt install nftables
sudo systemctl enable nftables
sudo systemctl start nftables
```

### 2\. Masquerading (NAT para a Internet)

```bash
# Criação de uma tabela NAT e regra de Masquerading
sudo nft add table ip nat
sudo nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
# Substitua 'eth0' pela sua interface WAN
sudo nft add rule ip nat postrouting oifname "eth0" masquerade
```

### 3\. Regras de Filtro (Requisitos Específicos)

A tabela principal de filtro (filter) deve aplicar as políticas de acesso.

| Origem | Destino | Serviço/Porta | Ação | Requisito |
| :--- | :--- | :--- | :--- | :--- |
| **LAN1 (10.16.100.0/24)** | LAN2 (172.16.254.0/24) | Qualquer | ACCEPT | Acesso irrestrito LAN1 -\> LAN2 |
| **LAN2 (172.16.254.0/24)** | LAN1 (10.16.100.0/24) | Qualquer | DROP | LAN2 mais restritiva |
| **LAN2 (172.16.254.0/24)** | WAN (eth0) | HTTP/S (80, 443), DNS (53) | ACCEPT | Acesso restrito à Internet (Exemplo) |

*Lembre-se de adicionar as regras para permitir o tráfego DNS/DHCP para o próprio servidor.*

-----

## 💾 III. Serviços de Rede (DHCP e DNS)

### 1\. Servidor DHCP (`isc-dhcp-server`)

```bash
sudo apt install isc-dhcp-server
# Edite /etc/default/isc-dhcp-server para definir as interfaces (eth1, eth2)
# Edite /etc/dhcp/dhcpd.conf para configurar os pools e o servidor DNS (10.16.100.1 e 172.16.254.1)

# Exemplo de Sub-rede LAN1 (em dhcpd.conf):
subnet 10.16.100.0 netmask 255.255.255.0 {
    range 10.16.100.100 10.16.100.200;
    option routers 10.16.100.1;
    option domain-name-servers 10.16.100.1; # O próprio servidor
}
```

> **Nota:** Use o **Webmin** (módulo **Servers \> DHCP Server**) para fixar endereços IP de forma amigável (Host Declarações).

### 2\. Servidor DNS (`bind9`)

```bash
sudo apt install bind9
# Edite /etc/bind/named.conf.options para configurar o Forwarding
options {
    forwarders {
        IP_DNS_OPERADORA_1;
        IP_DNS_OPERADORA_2;
    };
    forward only;
    // ... outras configurações ...
};
```

> **Objetivo:** O servidor escutará na porta 53 das interfaces LAN, responderá a consultas locais e encaminhará as externas para o DNS da operadora WAN.

-----

## 🔄 IV. Proxy Reverso (`nginx`)

Instale o NGINX e configure o host virtual para encaminhar as requisições para a VM de destino (e.g., uma máquina no 172.16.254.0/24).

```bash
sudo apt install nginx
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
```

-----

## 🔎 V. Relatório de Acesso Web (`Squid` e Relatório)

Para ter um relatório detalhado de sites visitados, use o Squid como Proxy **Explícito** (não reverso).

```bash
sudo apt install squid lightsquid apache2 # Lightsquid requer Apache/outro webserver

# 1. Configurar o Squid:
# Edite /etc/squid/squid.conf para definir as ACLs de acesso (permitindo LAN1 e LAN2)
# Configure o Squid para logar os acessos.
# A porta padrão é 3128.

# 2. Configurar o Lightsquid:
# Configure o lightsquid para ler os logs do squid e gerar os relatórios HTML.
# Configurar o Apache para servir o Lightsquid em uma porta/url restrita.
```

> **Aviso:** Os clientes de rede (LAN1 e LAN2) deverão ser configurados (manualmente ou via DHCP/GPO) para usar o servidor **10.16.100.1:3128** como proxy.

-----

## 🔑 VI. VPN Ponto-a-Ponto (`WireGuard`)

### 1\. Instalação e Chaves

```bash
sudo apt install wireguard
umask 077 
wg genkey | tee privatekey | wg pubkey > publickey # Gerar chaves
```

### 2\. Configuração do Servidor (`/etc/wireguard/wg0.conf`)

```conf
[Interface]
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
```

### 3\. Acesso Transparente (RDP/3389)

Com o WireGuard configurado acima, seu notebook (IP `10.0.1.2`) pode acessar a VM `172.16.254.10` na porta 3389 **de forma transparente**, desde que você **crie uma regra de firewall** para permitir o tráfego da rede VPN para a LAN2.

**Regra `nftables` (Exemplo):**

```bash
# Permitir conexões RDP da rede VPN (10.0.1.0/24) para LAN2 (172.16.254.0/24)
sudo nft add rule ip filter forward ip saddr 10.0.1.0/24 ip daddr 172.16.254.0/24 tcp dport 3389 counter accept
```

-----

## 🖥️ VII. Gerenciamento Web (`Webmin`)

O Webmin facilita a configuração do DHCP (fixar IPs), DNS e, em menor grau, o Firewall (via módulo Netfilter/iptables Legacy, mas **nftables** é preferível por CLI/Arquivo).

```bash
# Siga a documentação para adicionar o repositório e instalar:
sudo apt install webmin

# Acesso: https://[IP_DO_SEU_SERVIDOR]:10000
```

-----

Que outras funções de rede você gostaria de restringir ou permitir no seu firewall `nftables`? Posso gerar um template de configuração inicial para você.