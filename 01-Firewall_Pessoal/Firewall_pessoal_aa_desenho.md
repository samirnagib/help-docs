
# Desenho da Rede

## Topologia de Rede <img src="assets/img/network-analytic-b.svg" width="20" height="20" style="vertical-align: middle; margin-right: 8px;" />   

__Tabela__

| Rede | Tipo | Descrição | Network | Mascara |
| :--- | :--- | :--- | :--- | :--- |
| WAN | Externa | Modem da Operadora | x.x.x.x | 255.255.255.0 |
| LAN1 | Interna | Doméstica | 25.10.100.0 | 255.255.255.0 |
| LAN2 | Interna | Hypervisor | 172.16.254.0 | 255.255.255.0 |

__Regras:__

 - __LAN1__ e __LAN2__ acessam a internet, sendo que a LAN2 mais restritiva;
 - __LAN1__ acessa a  __LAN2__ sem restrição;
 - __LAN2__ acessa a  __LAN1__ em alguns serviços