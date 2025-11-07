Perfeito, vamos montar um exemplo completo de configuração do **Bind9** no Ubuntu Server 24.04.3 LTS para atender ao seu cenário.  

Você tem três interfaces, mas o Bind9 vai atuar como servidor DNS interno para as redes **lan1 (10.10.100.0/24)** e **lan2 (172.16.254.0/24)**, permitindo que elas façam consultas recursivas para a internet via **eno1 (WAN)**. Além disso, você quer criar zonas primárias e reversas para cada rede.

---

## 🛠️ Passos de Configuração

### 1. Instalar Bind9
```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc -y
```

---

### 2. Configurar `/etc/bind/named.conf.options`
Este arquivo define opções globais, como encaminhamento de consultas externas.

```bash
options {
    directory "/var/cache/bind";

    recursion yes;              // Permite consultas recursivas
    allow-recursion { 10.10.100.0/24; 172.16.254.0/24; }; // Apenas LANs internas
    allow-query { any; };       // Permite consultas

    forwarders {
        8.8.8.8;    // Google DNS
        1.1.1.1;    // Cloudflare DNS
    };

    dnssec-validation auto;

    listen-on { any; };
    listen-on-v6 { any; };
};
```

---

### 3. Configurar `/etc/bind/named.conf.local`
Aqui você define as zonas primárias e reversas.

```bash
// Zona LAN1 - home.local
zone "home.local" {
    type master;
    file "/etc/bind/db.home.local";
};

// Zona reversa LAN1
zone "100.10.10.in-addr.arpa" {
    type master;
    file "/etc/bind/db.10.10.100";
};

// Zona LAN2 - home.priv
zone "home.priv" {
    type master;
    file "/etc/bind/db.home.priv";
};

// Zona reversa LAN2
zone "254.16.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.172.16.254";
};
```

---

### 4. Criar os arquivos de zona

#### 📌 `/etc/bind/db.home.local`
```bash
$TTL    604800
@       IN      SOA     ns1.home.local. admin.home.local. (
                        2024110701 ; Serial
                        604800     ; Refresh
                        86400      ; Retry
                        2419200    ; Expire
                        604800 )   ; Negative Cache TTL

; Servidores
@       IN      NS      ns1.home.local.

; Registros
ns1     IN      A       10.10.100.1
pc1     IN      A       10.10.100.10
router  IN      A       10.10.100.254
```

---

#### 📌 `/etc/bind/db.10.10.100`
```bash
$TTL    604800
@       IN      SOA     ns1.home.local. admin.home.local. (
                        2024110701
                        604800
                        86400
                        2419200
                        604800 )

@       IN      NS      ns1.home.local.

1       IN      PTR     ns1.home.local.
10      IN      PTR     pc1.home.local.
254     IN      PTR     router.home.local.
```

---

#### 📌 `/etc/bind/db.home.priv`
```bash
$TTL    604800
@       IN      SOA     ns1.home.priv. admin.home.priv. (
                        2024110701
                        604800
                        86400
                        2419200
                        604800 )

@       IN      NS      ns1.home.priv.

ns1     IN      A       172.16.254.1
srv1    IN      A       172.16.254.10
router  IN      A       172.16.254.254
```

---

#### 📌 `/etc/bind/db.172.16.254`
```bash
$TTL    604800
@       IN      SOA     ns1.home.priv. admin.home.priv. (
                        2024110701
                        604800
                        86400
                        2419200
                        604800 )

@       IN      NS      ns1.home.priv.

1       IN      PTR     ns1.home.priv.
10      IN      PTR     srv1.home.priv.
254     IN      PTR     router.home.priv.
```

---

### 5. Validar e reiniciar Bind9
```bash
sudo named-checkconf
sudo named-checkzone home.local /etc/bind/db.home.local
sudo named-checkzone 100.10.10.in-addr.arpa /etc/bind/db.10.10.100
sudo named-checkzone home.priv /etc/bind/db.home.priv
sudo named-checkzone 254.16.172.in-addr.arpa /etc/bind/db.172.16.254

sudo systemctl restart bind9
sudo systemctl enable bind9
```

---

### 6. Testar
Do lado dos clientes (lan1 e lan2), configure o IP do servidor DNS (seu firewall/servidor Bind9).  
Testes:

```bash
dig @10.10.100.1 pc1.home.local
dig @172.16.254.1 srv1.home.priv
dig google.com
```

---

✅ Com isso, você terá:
- Resolução interna para **home.local** e **home.priv**.  
- Resolução reversa para ambas as redes.  
- Encaminhamento de consultas externas via **eno1** para DNS públicos.  

---

Quer que eu monte também um **diagrama lógico da arquitetura** (interfaces, fluxo de DNS interno/externo) para visualizar melhor como o Bind9 se encaixa no firewall?
