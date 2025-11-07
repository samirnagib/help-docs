### Visão geral
Um proxy transparente redireciona requisições HTTP dos clientes para o Squid sem precisar configurar o proxy em cada máquina. Você instala Squid (o LightSquid é só um gerador de relatórios usando os logs do Squid) e ajusta regras de encaminhamento/NAT no firewall para mandar o tráfego da porta 80 para a porta do Squid (normalmente 3128).

--- 

### Passo 1 — Instalar pacotes (Debian/Ubuntu)
1. Atualize e instale os pacotes:
```bash
sudo apt update
sudo apt install squid lightsquid apache2 -y
```
2. Habilite/ative serviços se necessário:
```bash
sudo systemctl enable --now squid
sudo systemctl enable --now apache2
```

---

### Passo 2 — Configurar o Squid (arquivo /etc/squid/squid.conf)
1. Edite /etc/squid/squid.conf e adicione / ajuste as linhas principais:
- Defina porta de interceptação:
```
http_port 3128 intercept
visible_hostname proxy-server
```
- ACLs básicas (ex.: permitir tudo — ajuste para seu ambiente):
```
acl Safe_ports port 80
acl CONNECT method CONNECT
http_access allow all
```
- Otimize cache/logs conforme desejar (cache_dir, cache_mem, access_log etc.).

2. Salve e reinicie o Squid:
```bash
sudo systemctl restart squid
```

Fonte: exemplos de configuração do Squid com modo intercept/transparente.

---

### Passo 3 — Habilitar encaminhamento IP (se servidor faz NAT)
Ative forwarding IPv4:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
# Para persistir:
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

---

### Passo 4 — Regras de firewall / iptables (redirecionar HTTP para Squid)
Exemplo típico em servidor que recebe o tráfego vindo da LAN (ajuste interfaces e IPs ao seu ambiente):

Se o Squid estiver na própria máquina gateway:
```bash
# redirecionar porta 80 para 3128
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3128

# permitir tráfego de saída/NAT
sudo iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
```

Se o Squid estiver em outro host (IP 192.168.1.10):
```bash
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.1.10:3128
```

Para firewalld use --add-forward-port ou regras equivalentes (ex.: firewall-cmd).

Depois de adicionar regras, teste acessando sites HTTP de um cliente; os logs do Squid (/var/log/squid/access.log) devem mostrar as requisições sendo atendidas.

---

### Passo 5 — LightSquid e Apache
- O LightSquid analisa /var/log/squid/access.log e gera relatórios HTML. Configure seu cron ou execute manualmente para gerar relatórios e coloque-os sob /var/www/html/ para servir via Apache.
- Exemplos de uso normalmente: ajustar o arquivo de configuração do LightSquid para apontar para o log do Squid e depois executar o script gerador. Apache só precisa servir o diretório dos relatórios.

---

### Observações importantes sobre HTTPS
- Redirecionar apenas o tráfego HTTP (porta 80) é simples. Interceptar HTTPS (porta 443) requer SSL Bump (man-in-the-middle) no Squid, geração/uso de uma CA própria e instalação dessa CA em todos os clientes para evitar erros de certificado. Isso tem implicações de privacidade e legais e geralmente é mais complexo e arriscado.
- Se você não quer interceptar HTTPS, permita que conexões HTTPS passem sem interceptação (não redirecione 443) ou use um proxy declarado (configurado nos clientes).

---

### Verificações e troubleshooting
- Verifique se o Squid está ouvindo com: sudo ss -tlpn | grep squid
- Confira logs: /var/log/squid/access.log e /var/log/squid/cache.log
- Teste do cliente: curl -v http://exemplo.com e observe se aparece no access.log do Squid
- Se usar iptables, salve regras (iptables-save) ou adicione via scripts/systemd para persistência.

---

### Referências rápidas
- Guia de configuração de Squid transparente e regras iptables/firewalld.
- Notas sobre interceptação HTTPS e necessidade de CA para SSL Bump.

Se quiser, eu preparo um exemplo de /etc/squid/squid.conf completo adaptado à sua rede (informe: IP do proxy, interfaces (LAN/WAN), se o proxy será gateway ou servidor separado, e se quer interceptar HTTPS).

Resumo da abordagem
Vou mostrar os passos e um conjunto pronto de comandos nftables + roteamento para interceptar HTTP (porta 80) com Squid rodando em modo intercept (http_port 3128 intercept). A solução usa TPROXY para preservar o destino original, marca pacotes e roteia tráfego marcado para o loopback, onde o Squid aceita as conexões. Recomendo adaptar as redes/ifs e a lista de endereços a excluir (bypass) para sua topologia.

Pré-requisitos
- Squid configurado com: http_port 3128 intercept e rodando.
- Kernel com suporte a TPROXY e módulos nft tproxy/socket carregáveis.
- Ajustes de sysctl conforme abaixo.

Ajustes de sysctl
Execute:
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.default.rp_filter=0
sudo sysctl -w net.ipv4.conf.lo.rp_filter=0
# para persistir:
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.conf.all.rp_filter=0" | sudo tee -a /etc/sysctl.conf



Roteamento para tráfego marcado (tabela local)
Crie uma tabela e regras de roteamento para enviar pacotes marcados ao loopback:
# criar tabela proxy_loopback (apenas uma vez)
echo "200 proxy_loopback" | sudo tee /etc/iproute2/rt_tables.d/proxy_loopback.conf

# adicionar regras e rotas
sudo ip rule add fwmark 0xC8 table proxy_loopback
sudo ip -6 rule add fwmark 0xC8 table proxy_loopback
sudo ip route add local 0.0.0.0/0 dev lo table proxy_loopback
sudo ip -6 route add local ::/0 dev lo table proxy_loopback


Aqui uso marca 0xC8 (200 decimal). Esta tabela força o kernel a encaminhar o tráfego marcado para o socket local onde o Squid recebe via TPROXY.

Exemplo nftables completo (ajuste interfaces e redes)
Substitua IIF (interface de entrada da LAN) e a porta do Squid se necessário. Salve como /etc/nftables.conf ou carregue com nft -f.
table inet proxy {
  set bypass4 {
    type ipv4_addr
    flags interval
    elements = {
      0.0.0.0/8,
      10.0.0.0/8,
      127.0.0.0/8,
      169.254.0.0/16,
      172.16.0.0/12,
      192.168.0.0/16,
      224.0.0.0/4,
      240.0.0.0/4
    }
  }

  chain prerouting {
    type filter hook prerouting priority -150; policy accept;
    # ignorar destinos locais/reservados
    ip daddr @bypass4 return

    # evitar capturar conexões originadas pelo próprio host
    iif "lo" return
    meta skgid 0 return

    # somente TCP destino 80 -> tproxy para porta 3128 e marcar
    tcp dport 80 tproxy to :3128 mark set 0xC8 accept
  }

  chain output {
    type route hook output priority -150; policy accept;
    ip daddr @bypass4 return
    # evitar marcar tráfego local ligado ao processo proxy
    meta skgid 0 return
    tcp dport 80 mark set 0xC8 accept
  }
}


Notas:
- A tabela "bypass4" contém redes a não interceptar (modifique conforme necessário).
- meta skgid/skuid podem ser usados para evitar interceptar conexões geradas pelo próprio processo do Squid (depende de kernel/versão).
- Se preferir, remova a cadeia output e use apenas prerouting quando o Squid for gateway.
Referências e detalhes sobre TPROXY/nftables e necessidade de marcar/rotear estão descritos em guias de nftables tproxy.

Comandos rápidos para aplicar sem arquivo
Se quiser testar imediatamente, execute (ajuste IIF, sets e porta se preciso):
# exemplo: interface LAN = eth0
sudo nft add table inet proxy
sudo nft 'add set inet proxy bypass4 { type ipv4_addr\; flags interval\; elements = { 0.0.0.0/8, 10.0.0.0/8, 127.0.0.0/8, 169.254.0.0/16, 172.16.0.0/12, 192.168.0.0/16 } }'
sudo nft 'add chain inet proxy prerouting { type filter hook prerouting priority -150; policy accept; }'
sudo nft add rule inet proxy prerouting ip daddr @bypass4 return
sudo nft add rule inet proxy prerouting iifname "lo" return
sudo nft add rule inet proxy prerouting tcp dport 80 tproxy to :3128 mark set 0xC8 accept



Verificações e resolução de problemas
- Confirme módulos: lsmod | grep -E "nf_tproxy|nft|nf_socket|nft_socket".
- Veja regras: sudo nft list ruleset.
- Logs do Squid: /var/log/squid/access.log para confirmar requisições interceptadas.
- Se match socket ou meta sk* não estiverem disponíveis, pode precisar dos módulos kmod-nf-socket / nft_socket ou recompilar kernel com suporte correspondente.
- Certifique-se de que Squid está ouvindo localmente na porta 3128 e com intercept habilitado.

Aviso sobre HTTPS
Esta configuração intercepta apenas HTTP (porta 80). Interceptar HTTPS exige SSL Bump e CA própria, com implicações legais/privacidade, e configurações adicionais de Squid e clientes.

Se quiser, adapto este exemplo à sua rede: me informe o nome da interface LAN (ex.: eth0), o IP do servidor proxy (se não for o gateway) e se precisa também interceptar UDP ou IPv6.


Resposta rápida: Para um proxy transparente com Squid em duas redes (10.10.100.0/24 e 172.16.254.0/24), você precisa configurar o Squid com portas http_port 3128 intercept e https_port 3129 intercept (para SSL Bump), gerar uma CA própria, habilitar encaminhamento IP, criar regras de roteamento com ip rule e ip route, e usar nftables com TPROXY para redirecionar tráfego HTTP (porta 80) e HTTPS (porta 443). Abaixo segue um documento detalhado com todos os passos.

1. Instalação de pacotes
sudo apt update
sudo apt install squid lightsquid apache2 openssl -y



2. Configuração do Squid
Edite /etc/squid/squid.conf:
# HTTP transparente
http_port 3128 intercept

# HTTPS transparente (SSL Bump)
https_port 3129 intercept ssl-bump cert=/etc/squid/ssl_cert/myCA.pem key=/etc/squid/ssl_cert/myCA.key

# ACLs básicas
acl localnet src 10.10.100.0/24
acl localnet src 172.16.254.0/24
http_access allow localnet
http_access deny all

# SSL Bump
acl step1 at_step SslBump1
ssl_bump peek step1
ssl_bump bump all

# Logs
access_log /var/log/squid/access.log



3. Geração da CA para SSL Bump
sudo mkdir -p /etc/squid/ssl_cert
cd /etc/squid/ssl_cert
sudo openssl req -new -newkey rsa:2048 -days 3650 -nodes -x509 \
  -keyout myCA.key -out myCA.pem
sudo chown -R proxy:proxy /etc/squid/ssl_cert


⚠️ É necessário instalar esse certificado nos clientes para evitar erros de HTTPS.

4. Ajustes de sysctl
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.default.rp_filter=0
sudo sysctl -w net.ipv4.conf.lo.rp_filter=0



5. Roteamento com ip rule/ip route
echo "200 proxy_loopback" | sudo tee /etc/iproute2/rt_tables.d/proxy_loopback.conf
sudo ip rule add fwmark 0xC8 table proxy_loopback
sudo ip route add local 0.0.0.0/0 dev lo table proxy_loopback



6. Regras nftables (HTTP + HTTPS)
Arquivo /etc/nftables.conf:
table inet proxy {
  set bypass4 {
    type ipv4_addr
    flags interval
    elements = {
      127.0.0.0/8,
      169.254.0.0/16,
      224.0.0.0/4,
      240.0.0.0/4
    }
  }

  chain prerouting {
    type filter hook prerouting priority -150; policy accept;
    ip daddr @bypass4 return
    iif "lo" return

    # HTTP
    tcp dport 80 tproxy to :3128 mark set 0xC8 accept

    # HTTPS
    tcp dport 443 tproxy to :3129 mark set 0xC8 accept
  }
}


Ative:
sudo nft -f /etc/nftables.conf



7. LightSquid + Apache
- Configure LightSquid para ler /var/log/squid/access.log.
- Gere relatórios com lightsquid.pl.
- Sirva relatórios via Apache em /var/www/html/lightsquid.

8. Verificações
- sudo systemctl status squid
- tail -f /var/log/squid/access.log
- sudo nft list ruleset
- Teste clientes acessando sites HTTP/HTTPS.

9. Avisos legais
- Interceptar HTTPS com SSL Bump é invasivo: você estará atuando como “man-in-the-middle”. Isso exige consentimento dos usuários e pode ter implicações legais e de privacidade.
- Em ambientes corporativos, distribua a CA para todos os clientes de forma controlada.

10. Troubleshooting
- Se HTTPS falhar, verifique se o certificado CA está instalado nos clientes.
- Confirme que Squid está ouvindo nas portas 3128 e 3129.
- Cheque se as regras nftables estão aplicadas corretamente.

👉 Posso preparar um exemplo de configuração de distribuição da CA nos clientes (Windows/Linux) para que o SSL Bump funcione sem erros de certificado. Quer que eu detalhe isso?

Resposta direta: Para que o SSL Bump funcione sem erros de certificado nos seus dispositivos, você precisa instalar a CA que gerou no Squid tanto em computadores quanto em celulares. Em Windows/macOS, a CA vai para o repositório de certificados confiáveis do sistema. Em Android/iOS, você precisa importar manualmente o arquivo .pem e confiar nele como certificado raiz.

📱 Instalação da CA em celulares
iPhone / iPad (iOS)
- Envie o arquivo myCA.pem para o dispositivo (por e‑mail, AirDrop ou iCloud Drive).
- Abra o arquivo no iPhone → ele será reconhecido como certificado.
- Toque em Instalar e confirme.
- Vá em Ajustes → Geral → Informações → Ajustes de confiança do certificado.
- Ative a confiança total para o certificado recém‑instalado.
⚠️ Sem esse passo final, o iOS não usará a CA para HTTPS.

Android
- Copie o arquivo myCA.pem para o celular.
- Vá em Configurações → Segurança → Instalar certificado → Armazenamento.
- Escolha o arquivo e instale como Certificado de CA confiável.
- Em alguns Androids, pode aparecer como “Credenciais confiáveis do usuário”.
⚠️ Se o Android estiver em modo corporativo (Work Profile), pode ser necessário instalar via MDM ou políticas de segurança.

💻 Instalação em desktops
Windows
- Clique com botão direito no arquivo myCA.pem → Instalar certificado.
- Escolha Computador Local.
- Selecione Colocar todos os certificados no repositório a seguir → Autoridades de Certificação Raiz Confiáveis.
- Conclua e confirme.
macOS
- Abra o arquivo myCA.pem → ele abrirá no Acesso às Chaves.
- Arraste para Sistema → Certificados.
- Clique duas vezes no certificado → em “Confiar”, marque Sempre confiar.

🔍 Verificação
- Acesse um site HTTPS pelo navegador do celular/computador.
- Se o Squid interceptar corretamente, o certificado apresentado será o da sua CA.
- Se aparecer erro de certificado, significa que a CA não foi instalada ou não está marcada como confiável.

⚠️ Aviso importante
- Interceptar HTTPS com SSL Bump é um man‑in‑the‑middle: você verá todo o tráfego criptografado dos usuários. Isso só deve ser feito em ambientes controlados (laboratórios, empresas com consentimento).
- Em celulares pessoais, instalar uma CA significa que o proxy pode decifrar qualquer conexão HTTPS — use com responsabilidade.

👉 Posso detalhar também como distribuir automaticamente esse certificado em larga escala (ex.: via MDM para iPhones ou Android Enterprise) se você pretende aplicar em vários dispositivos da rede. Quer que eu monte esse guia corporativo?
