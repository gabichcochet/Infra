## PROJET INFRA

Architecture

1. VM-ROUTER
OS : Debian

Pourquoi Debian ?
-Très léger → parfait pour un routeur minimaliste
-Stabilité exemplaire → un routeur ne doit jamais planter
-Configuration réseau simple et standard
-Documentation immense pour : iptables, nftables, NAT, routage, DHCP, VPN
-Utilisé comme base dans de nombreuses appliances réseau (VyOS, IPFire, Untangle)

2. VM-SRV-DNS
OS : Debian

Pourquoi Debian ?
-Bind9 est historiquement mieux documenté sur Debian
-Très stable, idéal pour un service d’infrastructure
-Peu gourmand en ressources
-Maintenance simple, mises à jour non intrusives
-Parfait pour un DNS interne d’entreprise


3. VM-SRV-WEB
OS : Debian

Pourquoi Debian ?
-Nginx/Apache sont extrêmement bien supportés
-Très léger → idéal pour une VM web
-Gestion simple des certificats HTTPS (Certbot)
-Compatible Docker si tu conteneurises ton service
-Très utilisé dans les environnements web professionnels

4. VM-SRV-DATA (Base de données)
OS : Rocky Linux

Pourquoi Rocky Linux ?
-Compatible RHEL → standard des entreprises
-Très robuste pour les services critiques (DB, stockage)
-Cycle de support long (10 ans)
-Optimisé pour MariaDB/PostgreSQL
-Très utilisé dans les datacenters et environnements VMware/Proxmox


5. VM-BACKUP
OS : Debian

Pourquoi Debian ?

-Très léger
-Parfait pour des outils comme Borg, Restic, Rsync
-Maintenance simple
-Idéal si tu veux un serveur de backup minimaliste

# VM Routeur — Configuration Complète

## Rôle de la VM Routeur
La VM Routeur assure la communication entre les différentes zones du réseau et Internet.  
Elle remplit quatre fonctions principales :

- Fournir l’accès Internet au LAN et à la DMZ via **NAT**
- Assurer le **routage** entre les réseaux internes
- Appliquer un **pare‑feu** pour contrôler les flux
- Isoler la **DMZ** du **LAN** pour des raisons de sécurité


## Configuration réseau

La VM routeur possède **4 interfaces réseau** :

| Interface | Type VirtualBox | Rôle | Adresse IP |
|----------|------------------|------|-------------|
| **enp0s3** | NAT | Accès Internet | DHCP (10.0.2.15) |
| **enp0s8** | Réseau interne | LAN | 192.168.10.1/24 |
| **enp0s9** | Réseau interne | DMZ | 192.168.20.1/24 |
| **enp0s10** | Host‑Only | Administration SSH depuis Windows | 192.168.56.1/24 |

### Configuration des interfaces réseaux dans `/etc/network/interfaces`

```bash
# WAN (NAT)
auto enp0s3
iface enp0s3 inet dhcp

# LAN
auto enp0s8
iface enp0s8 inet static
    address 192.168.10.1
    netmask 255.255.255.0

# DMZ
auto enp0s9
iface enp0s9 inet static
    address 192.168.20.1
    netmask 255.255.255.0

# Host-Only (SSH depuis Windows)
auto enp0s10
iface enp0s10 inet static
    address 192.168.56.1
    netmask 255.255.255.0
```


### Activation de la config

```bash
root@ROUTEUR:/home/gabriel# systemctl restart networking
```

### Activation du routage IP dans `/etc/sysctl.conf`

```bash
net.ipv4.ip_forward = 1
```

### Application 

```bash
root@ROUTEUR:/home/gabriel# sudo sysctl -p
```

### Pare-feu iptables


```bash
root@ROUTEUR:/home/gabriel# nano /etc/iptables/rules.v4
```

```bash
*filter

# Politiques par défaut
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]

# Règles de base
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT

# SSH depuis Host-Only
-A INPUT -i enp0s10 -p tcp --dport 22 -j ACCEPT
-A INPUT -i enp0s10 -j DROP

# LAN <-> Internet
-A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
-A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT

# DMZ <-> Internet
-A FORWARD -i enp0s9 -o enp0s3 -j ACCEPT
-A FORWARD -i enp0s3 -o enp0s9 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Isolation DMZ <-> LAN
-A FORWARD -i enp0s9 -o enp0s8 -j DROP
-A FORWARD -i enp0s8 -o enp0s9 -j DROP

COMMIT
```

### Sauvegarde des règles

```bash
root@ROUTEUR:/home/gabriel# sudo netfilter-persistent save
```

```bash
root@ROUTEUR:/home/gabriel# sudo netfilter-persistent reload
```

### Vérification totale

```bash
root@ROUTEUR:/home/gabriel# iptables -t nat -L -v -n
```

```bash 
root@ROUTEUR:/home/gabriel# systemctl status netfilter-persistent
```

# 🖧 VM SERVEUR DNS — Configuration Complète

## Rôle de la VM DNS 

La VM VMDNSSERV fournit le service DNS interne pour l’infrastructure.
Elle remplit trois fonctions principales :

- Héberger un serveur Bind9 authoritative pour le domaine interne infra.local
- Fournir la résolution interne des machines du LAN
- Offrir la résolution inverse pour le réseau 192.168.10.0/24
- Cette VM est essentielle : elle permet aux autres machines de communiquer via des noms     plutôt que des adresses IP.

## Configuration réseau

La VM routeur possède **2 interfaces réseau** :

| Interface | Type VirtualBox | Rôle | Adresse IP |
|----------|------------------|------|-------------|
| **enp0s3** | Réseau interne | LAN | 192.168.10.5/24 |
| **enp0s9** | Host‑Only | Administration SSH depuis Windows | DHCP(192.168.56.104/24) |

### Configuration des interfaces dans `/etc/network/interfaces`

```bash
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# Interface LAN (vers le routeur)
auto enp0s3
iface enp0s3 inet static
    address 192.168.10.5
    netmask 255.255.255.0
    gateway 192.168.10.1
    dns-nameservers 127.0.0.1

# Interface Host-Only (SSH)
auto enp0s9
iface enp0s9 inet dhcp
```


```bash
root@DNSERV:/home/gabriel# systemctl restart networking
```


### Configuration du serveur DNS


#### Installer Bind9 si c'est pas déjà fait 


```bash
options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { 127.0.0.1; 192.168.10.0/24; };

    forwarders {
        1.1.1.1;
        8.8.8.8;
    };

    dnssec-validation auto;

    listen-on { 127.0.0.1; 192.168.10.5; };
    listen-on-v6 { none; };
};
```
#### Fichier /etc/bind/named.conf.local

```bash
//
// Do any local configuration here
//
zone "infra.local" {
    type master;
    file "/etc/bind/db.infra.local";
};

zone "10.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.10";
};
```

#### Zone directe : /etc/bind/db.infra.local

```bash
$TTL 86400
@   IN  SOA dns.infra.local. admin.infra.local. (
        2024011901
        3600
        1800
        604800
        86400 )

@       IN  NS      dns.infra.local.
dns     IN  A       192.168.10.5
web     IN  A       192.168.10.20
db      IN  A       192.168.10.30
```

#### Zone inverse : /etc/bind/db.192.168.10

```bash
$TTL 86400
@   IN  SOA dns.infra.local. admin.infra.local. (
        2024011901
        3600
        1800
        604800
        86400 )

@       IN  NS      dns.infra.local.
10      IN  PTR     dns.infra.local.
20      IN  PTR     web.infra.local.
30      IN  PTR     db.infra.local.
```

#### Vérifier la configuration


```bash
root@DNSERV:/home/gabriel# named-checkconf
root@DNSERV:/home/gabriel# named-checkzone infra.local /etc/bind/db.infra.local
zone infra.local/IN: loaded serial 2024011901
OK
root@DNSERV:/home/gabriel# named-checkzone 10.168.192.in-addr.arpa /etc/bind/db.192.168.10
zone 10.168.192.in-addr.arpa/IN: loaded serial 2024011901
OK
```

#### Redémarrer Bind9

```bash
root@DNSERV:/home/gabriel# systemctl restart bind9
systemctl status bind9
● named.service - BIND Domain Name Server
     Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-01-21 15:06:40 CET; 86ms ago
 Invocation: 846dc2de05c3429594a558a1211a1427
       Docs: man:named(8)
   Main PID: 2838 (named)
     Status: "running"
      Tasks: 10 (limit: 3116)
     Memory: 43M (peak: 43.2M)
        CPU: 508ms
     CGroup: /system.slice/named.service
             └─2838 /usr/sbin/named -f -u bind

janv. 21 15:06:40 DNSERV named[2838]: network unreachable resolving './NS/IN': 2001:7fe::53#53
janv. 21 15:06:40 DNSERV named[2838]: network unreachable resolving './NS/IN': 2001:500:1::53#53
janv. 21 15:06:40 DNSERV named[2838]: network unreachable resolving './NS/IN': 2001:500:2::c#53
janv. 21 15:06:40 DNSERV named[2838]: network unreachable resolving './NS/IN': 2001:dc3::35#53
janv. 21 15:06:40 DNSERV named[2838]: network unreachable resolving './NS/IN': 2001:500:2f::f#53
janv. 21 15:06:40 DNSERV named[2838]: network unreachable resolving './NS/IN': 2001:503:c27::2:30#53
janv. 21 15:06:40 DNSERV named[2838]: network unreachable resolving './NS/IN': 2001:500:9f::42#53
janv. 21 15:06:40 DNSERV named[2838]: network unreachable resolving './NS/IN': 2001:503:ba3e::2:30#53
janv. 21 15:06:40 DNSERV named[2838]: network unreachable resolving './NS/IN': 2001:7fd::1#53
janv. 21 15:06:40 DNSERV named[2838]: network unreachable resolving './NS/IN': 2001:500:2d::d#53
```


#### Vérifier que Bind9 écoute


```bash 
root@DNSERV:/home/gabriel# ss -tulpn | grep :53
```

# 🖧 VM SERVEUR WEB — Configuration Complète

## Rôle de la VM SRVWEB
La VM SRVWEB héberge le serveur web interne de l’infrastructure.
Elle assure plusieurs fonctions essentielles :

Fournir un service HTTP accessible depuis tout le LAN interne

Héberger les pages web du projet (site statique ou dynamique)

Servir de point d’accès applicatif pour les autres machines

Être administrée facilement via SSH grâce à une interface Host‑Only

Cette VM est un élément clé de l’infrastructure : elle permet d’exposer un service web interne fiable et accessible via le DNS.

## Configuration réseau

La VM routeur possède **2 interfaces réseau** :

| Interface | Type VirtualBox | Rôle | Adresse IP |
|----------|------------------|------|-------------|
| **enp0s3** | Réseau interne | LAN | 192.168.10.20/24|
| **enp0s9** | Host‑Only | Administration SSH depuis Windows | DHCP (192.168.56.109) |


## Configuration des interfaces dans `/etc/network/interfaces` 


```bash
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# Interface LAN (vers le routeur)
auto enp0s3
iface enp0s3 inet static
    address 192.168.10.20
    netmask 255.255.255.0
    gateway 192.168.10.1
    dns-nameservers 192.168.10.5

# Interface Host-Only (SSH)
auto enp0s9
iface enp0s9 inet dhcp
```

```bash
root@SERVWEB:/# systemctl restart networking
```

### Installer Apache 

```bash
root@SERVWEB:/# apt install apache2
```

### Vérification du service

```bash
root@SERVWEB:/# systemctl status apache2
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/apache2.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-01-21 15:52:52 CET; 32min ago
 Invocation: 9bcc03a93c42463eb347e22ab6ed4490
       Docs: https://httpd.apache.org/docs/2.4/
    Process: 2580 ExecReload=/usr/sbin/apachectl graceful (code=exited, status=0/SUCCESS)
   Main PID: 2228 (apache2)
      Tasks: 55 (limit: 3116)
     Memory: 6.1M (peak: 8.5M)
        CPU: 834ms
     CGroup: /system.slice/apache2.service
             ├─2228 /usr/sbin/apache2 -k start
             ├─2584 /usr/sbin/apache2 -k start
             └─2613 /usr/sbin/apache2 -k start

janv. 21 15:52:52 SERVWEB systemd[1]: Starting apache2.service - The Apache HTTP Server...
janv. 21 15:52:52 SERVWEB apachectl[2227]: AH00558: apache2: Could not reliably determine the server's fully qualified >
janv. 21 15:52:52 SERVWEB systemd[1]: Started apache2.service - The Apache HTTP Server.
janv. 21 15:54:06 SERVWEB systemd[1]: Reloading apache2.service - The Apache HTTP Server...
janv. 21 15:54:06 SERVWEB apachectl[2583]: AH00558: apache2: Could not reliably determine the server's fully qualified >
janv. 21 15:54:06 SERVWEB systemd[1]: Reloaded apache2.service - The Apache HTTP Server.
lines 1-21/21 (END)
```

### Déploiement du site web

```bash
root@SERVWEB:/# echo "<h1>Bienvenue sur SRV WEB</h1>" > /var/www/html/index.html
```

### Configuration DNS pour SRVWEB

Dans la zone DNS (hébergée sur DNSERV), SRVWEB est déclaré ainsi :

```bash
web     IN  A   192.168.10.20
```

### Configuration DNS

Rajouter dans `nano /etc/resolv.conf.head` de la VM routeur et la Web : 

```bash
nameserver 192.168.10.5
```

### Sécurisation minimale d’Apache


```bash
root@SERVWEB:/# nano /etc/apache2/conf-available/security.conf
```

Modifier :

```bash
ServerTokens 
ServerSignature 
```

pour :

```bash
ServerTokens Prod
ServerSignature Off
```

Puis recharger : 

```bash
root@SERVWEB:/# systemctl reload apache2
```

### Tests finaux

```bash
root@SERVWEB:/# ping dns.infra.local
PING dns.infra.local (192.168.10.5) 56(84) bytes of data.
64 bytes from 192.168.10.5: icmp_seq=1 ttl=64 time=5.86 ms
64 bytes from 192.168.10.5: icmp_seq=2 ttl=64 time=2.01 ms
^C
--- dns.infra.local ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1006ms
rtt min/avg/max/mdev = 2.010/3.933/5.857/1.923 ms
root@SERVWEB:/#
```

```bash
root@SERVWEB:/# ping web.infra.local
PING web.infra.local (192.168.10.20) 56(84) bytes of data.
64 bytes from SERVWEB (192.168.10.20): icmp_seq=1 ttl=64 time=0.051 ms
64 bytes from SERVWEB (192.168.10.20): icmp_seq=2 ttl=64 time=0.051 ms
^C
--- web.infra.local ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1026ms
rtt min/avg/max/mdev = 0.051/0.051/0.051/0.000 ms
root@SERVWEB:/#
```


```bash
root@SERVWEB:/# curl http://web.infra.local
<h1>Bienvenue sur SRV WEB</h1>
```

### Vérification du LAN

```bash
root@SERVWEB:/# ping 192.168.10.5
PING 192.168.10.5 (192.168.10.5) 56(84) bytes of data.
64 bytes from 192.168.10.5: icmp_seq=1 ttl=64 time=3.08 ms
64 bytes from 192.168.10.5: icmp_seq=2 ttl=64 time=3.21 ms
^C
--- 192.168.10.5 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
rtt min/avg/max/mdev = 3.081/3.147/3.213/0.066 ms
```