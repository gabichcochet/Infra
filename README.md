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
OS : Debian

Pourquoi Debian ?
-MariaDB est parfaitement supporté
-Debian applique des politiques de sécurité strictes, parfait pour un serveur SQL.
-Cohérence avec tes autres VMs

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

- Fournir l’accès Internet au LAN via **NAT**
- Assurer le **routage** entre les réseaux internes
- Appliquer un **pare‑feu** pour contrôler les flux
- Isoler la **LAN** pour des raisons de sécurité


## Configuration réseau

La VM routeur possède **4 interfaces réseau** :

| Interface | Type VirtualBox | Rôle | Adresse IP |
|----------|------------------|------|-------------|
| **enp0s3** | NAT | Accès Internet | DHCP (10.0.2.15) |
| **enp0s8** | Réseau interne | LAN | 192.168.10.1/24 |
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

# VM SERVEUR DNS — Configuration Complète

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

# VM SERVEUR WEB — Configuration Complète

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
| **enp0s9** | Host‑Only | Administration SSH depuis Windows | 192.168.56.109 |


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

# VM SERVEUR DE BASE DE DONNÉES — Configuration Complète


## Rôle de la VM SRVDB

La VM SRVDB héberge le serveur de base de données MariaDB de l’infrastructure.
Elle assure plusieurs fonctions essentielles :

Fournir un service SQL fiable pour les applications hébergées sur SRVWEB

Centraliser les données du projet dans une base unique (infra_db)

Garantir un accès sécurisé grâce à un utilisateur dédié (webuser)

Limiter l’accès SQL uniquement au serveur web (192.168.10.20)

Être administrée facilement via SSH grâce à une interface Host‑Only

Cette VM est un élément critique de l’infrastructure : elle stocke les données et assure la cohérence applicative.

## Configuration réseau

### La VM SRVDB possède 2 interfaces réseau :

| Interface | Type VirtualBox | Rôle | Adresse IP |
|----------|------------------|------|-------------|
| **enp0s3** | Réseau interne | LAN | 192.168.10.30/24|
| **enp0s9** | Host‑Only | Administration SSH depuis Windows | DHCP (192.168.56.114) |

### Configuration des interfaces dans /etc/network/interfaces

```bash
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# Interface LAN (vers le routeur)
auto enp0s3
iface enp0s3 inet static
    address 192.168.10.30
    netmask 255.255.255.0
    gateway 192.168.10.1

# Interface Host-Only (SSH)
auto enp0s9
iface enp0s9 inet dhcp
```

```bash
root@VMSERVDATA:/# systemctl restart networking
```

### Configuration DNS

Rajouter dans `nano /etc/resolv.conf.head` de la VM Data : 

```bash
nameserver 192.168.10.5
```

### Installation de MariaDB Server

```bash
root@VMSERVDATA:/# apt install mariadb-server
```

### Vérification du service :

```bash
root@VMSERVDATA:/# systemctl status mariadb
```

### Configuration de l’écoute réseau (bind-address) :

```bash
root@VMSERVDATA:/# nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

### Éditer `nano /etc/mysql/mariadb.conf.d/50-server.cnf`:

Modifier : 

```bash
bind-address = 127.0.0.1
```

pour : 

```bash
bind-address = 192.168.10.30
```

Redémarrer :

```bash
root@VMSERVDATA:/# systemctl restart mariadb
```

Vérification :

```bash
root@VMSERVDATA:/home/gabriel#  ss -tulpn | grep 3306
tcp   LISTEN 0      80                          192.168.10.30:3306       0.0.0.0:*    users:(("mariadbd",pid=6288,fd=28))
```

## Sécurisation de MariaDB

Définir un mot de passe root :

```bash
ALTER USER 'root'@'localhost' IDENTIFIED BY '1234';
FLUSH PRIVILEGES;
```

Supprimer les utilisateurs anonymes :

```bash
DELETE FROM mysql.user WHERE User='';
FLUSH PRIVILEGES;
```

Interdire l’accès root distant :

```bash
DELETE FROM mysql.user WHERE User='root' AND Host!='localhost';
FLUSH PRIVILEGES;
```

Supprimer la base de test :

```bash
DROP DATABASE IF EXISTS test;
FLUSH PRIVILEGES;
```

## Création de la base et de l’utilisateur applicatif

Connexion :

```bash
mysql -u root -p
```


Créer la base :


```bash
CREATE DATABASE infra_db;
```

Créer l’utilisateur dédié à SRVWEB :


```bash
CREATE USER 'webuser'@'192.168.10.20' IDENTIFIED BY 'MotDePasseWeb';
GRANT ALL PRIVILEGES ON infra_db.* TO 'webuser'@'192.168.10.20';
FLUSH PRIVILEGES;
```

Vérification :

```bash
SELECT User, Host FROM mysql.user;
```

## Configuration du firewall UFW 

La politique par défaut est DROP, donc il faut autoriser SRVWEB.
Autoriser le port SQL uniquement pour SRVWEB :

```bash
root@VMSERVDATA:/# ufw allow from 192.168.10.20 to any port 3306 proto tcp
```

```bash
root@VMSERVDATA:/# ufw status numbered
```

## Test de connexion depuis SRVWEB

```bash
root@SERVWEB:/# mysql -h 192.168.10.30 -u webuser -p
```

```bash
root@SERVWEB:/home/gabriel# mysql -h 192.168.10.30 -u webuser -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 20
Server version: 11.8.3-MariaDB-0+deb13u1 from Debian -- Please help get to 10k stars at https://github.com/MariaDB/Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

# Infrastructure de Sauvegarde (SRVWEB / SRVDB → VMBACKUP)

Cette infrastructure met en place un système de sauvegarde automatisé pour :

SRVWEB : sauvegarde des fichiers du site web (/var/www/html)
SRVDB : sauvegarde de la base de données MariaDB (infra_db)
VMBACKUP : serveur central de stockage des sauvegardes

Les sauvegardes sont transférées via SSH sécurisé, sans mot de passe, et exécutées automatiquement via cron.

## Configuration réseau

| Interface | Type VirtualBox | Rôle | Adresse IP |
|----------|------------------|------|-------------|
| **enp0s3** | Réseau interne | LAN | 192.168.10.40/24|
| **enp0s8** | Host‑Only | Administration SSH depuis Windows | DHCP (192.168.56.116) |



```bash
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# Interface LAN (vers le routeur)
auto enp0s3
iface enp0s3 inet static
    address 192.168.10.40
    netmask 255.255.255.0
    gateway 192.168.10.1
    dns-nameservers 192.168.10.5

# Interface Host-Only (SSH)
auto enp0s8
iface enp0s8 inet dhcp
```

```bash
root@SRVBACKUP:~# systemctl restart networking
```


### Installer rsync 

```bash
root@SRVBACKUP:~# apt install rsync openssh-server
```

### Arborescence de sauvegarde

```bash
root@SRVBACKUP:/# mkdir -p /backup/srvweb
root@SRVBACKUP:/# mkdir -p /backup/srvdb
root@SRVBACKUP:/# mkdir -p /backup/configs
chmod -R 700 /backup
```

### Création d’un utilisateur dédié aux sauvegardes

```bash
root@SRVBACKUP:/# adduser backup
root@SRVBACKUP:/# passwd backup
```

### Script de sauvegarde depuis SRVWEB

Sur SRVWEB, créer :

```bash
nano /usr/local/bin/backup_web.sh
```

Contenu : 

```bash
#!/bin/bash
echo "Backup web lancé à $(date)" >> /var/log/backup_web.log
rsync -avz /var/www/html/ backup@192.168.10.40:/backup/srvweb/
```

Rendre exécutable :

```bash
chmod +x /usr/local/bin/backup_web.sh
```


### Script de sauvegarde SQL depuis SRVDB

Sur SRVDB, créer :

```bash
nano /usr/local/bin/backup_db.sh
```

Contenu :

```bash
#!/bin/bash
echo "Backup lancé à $(date)" >> /var/log/backup_db.log
mysqldump -u root -p'TonMotDePasse' infra_db > /tmp/infra_db.sql
rsync -avz /tmp/infra_db.sql backup@192.168.10.40:/backup/srvdb/
rm /tmp/infra_db.sql
```

Rendre exécutable :

```bash
chmod +x /usr/local/bin/backup_db.sh
```

### Automatisation via cron 

Sur SRVWEB :

```bash
crontab -e
```

```bash
0 2 * * * /usr/local/bin/backup_web.sh
```

Sur SRVDB :

```bash
crontab -e
```

```bash
0 3 * * * /usr/local/bin/backup_db.sh
```

Assurer vous que les connexion SSH depuis les VM Web et VM DATA qu'elles puissent accéder à la VM Backup sans demande du mot de passe :

Générer une clé SSH sur SRVDB :

```bash
ssh-keygen -t ed25519 -C "backup-db"
```

Quand il est demandé un mot de passe tapez "entrée" pour rendre le mot de passe vide.


```bash
ssh-copy-id backup@192.168.10.40
```

Tester le :

```bash
ssh backup@192.168.10.40
```

Générer une clé SSH sur SRVWEB :

```bash
ssh-keygen -t ed25519 -C "backup-web"
```

Quand il est demandé un mot de passe tapez "entrée" pour rendre le mot de passe vide.


```bash
ssh-copy-id backup@192.168.10.40
```

### Créer le dossier .ssh sur la VM BACKUP

```bash
mkdir -p /var/backups/.ssh
chown backup:backup /var/backups/.ssh
chmod 700 /var/backups/.ssh
```


### Tests finaux

Sur la VM WEB :

```bash
root@VMBACKUP:/home/gabriel# rm /backup/srvweb/index.html
```

```bash
root@VMBACKUP:/home/gabriel# ls -lh /backup/srvweb/
total 8,0K
-rw-r--r-- 1 backup backup 31 21 janv. 15:54 index.html
-rw-rw-r-- 1 backup backup 47 21 janv. 16:28 index.htmlping
```

Sur la VM DATA :

```bash
root@VMBACKUP:/home/gabriel# rm /backup/srvdb/infra_db.sql
```


```bash
root@VMBACKUP:/home/gabriel# ls -lh /backup/srvdb/
total 12K
-rw-r--r-- 1 backup backup   38 10 déc.  09:16 index.html
-rw-r--r-- 1 backup backup  615  8 déc.  11:28 index.nginx-debian.html
-rw-r--r-- 1 backup backup 2,1K 22 janv. 20:19 infra_db.sql
```

# Conteneurisation sur la VM BACKUP

## Installation de Docker

```bash
curl -fsSL https://get.docker.com | sh
usermod -aG docker root
```

```bash
docker version
docker run hello-world
```


## Mise en place de Restic (conteneurisé)

```bash
mkdir -p /srv/backup/{data,config,logs}
```

```bash
echo "changeme" > /srv/backup/config/restic.pass
chmod 600 /srv/backup/config/restic.pass
```

```bash
docker-compose.yml
```

```bash
services:
restic:
image: restic/restic:latest
volumes:
- ./data:/backup
- /backup/srvweb:/data-srvweb:ro
- /backup/srvdb:/data-srvdb:ro
- ./config:/config:ro
environment:
RESTIC_REPOSITORY: /backup
RESTIC_PASSWORD_FILE: /config/restic.pass
```

## Initialisation du dépôt Restic

```bash
cd /srv/backup
docker compose run --rm restic init
```

## Script de sauvegarde automatisée

```bash
nano /srv/backup/config/backup.sh
```

```bash
#!/bin/bash
set -e


DATE=$(date +"%Y-%m-%d_%H-%M")
LOG_DIR="/srv/backup/logs"
LOG_FILE="$LOG_DIR/backup_$DATE.log"


mkdir -p "$LOG_DIR"
cd /srv/backup


echo "=== Backup started at $(date) ===" >> "$LOG_FILE"


/usr/bin/docker compose run --rm restic backup \
/data-srvweb \
/data-srvdb >> "$LOG_FILE" 2>&1


/usr/bin/docker compose run --rm restic forget \
--keep-daily 7 \
--keep-weekly 4 \
--keep-monthly 6 \
--prune >> "$LOG_FILE" 2>&1

echo "=== Backup finished at $(date) ===" >> "$LOG_FILE"
```

```bash
chmod +x /srv/backup/config/backup.sh
```

## Automatisation via cron

```bash
crontab -e
```

contenu : 

```bash
*/1 * * * * /srv/backup/config/backup.sh
```


## Tests de restauration

```bash
docker compose run --rm restic snapshots
```

```bash
docker compose run --rm restic restore latest --target /backup/restore-test
```

Vérification :

```bash
find /backup/restore-test
```

Vérification des logs :

```bash
tail -n 50 /srv/backup/logs/backup_*.log
```

# Sauvegarde centralisée automatisée


Architecture Logique

```bash
SRVWEB  ---- rsync ----\
                        -->  VMBACKUP  --> Restic (Docker) --> snapshots
SRVDB   ---- rsync ----/
```

Arborescence principale sur VMBACKUP


```bash
/backup/
├── srvweb/        # données web reçues
└── srvdb/         # dumps SQL reçus

/srv/backup/
├── docker-compose.yml
├── config/
│   ├── restic.pass
│   └── backup.sh
├── data/          # dépôt Restic
└── logs/          # logs de sauvegarde
```


## Automatisation avec Ansible

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

```bash
mkdir -p ansible/{roles/{common,srvweb,srvdb,vmbackup}/{tasks,files},group_vars}
cd ansible
```


Crée Ansible.cfg dans `/srv/backup/data/ansible` :

```bash
[defaults]
inventory = inventory.ini
host_key_checking = False
interpreter_python = auto_silent
retry_files_enabled = False

[ssh_connection]
pipelining = True
```

Crée inventory.ini (adapte IP + user SSH) dans `/srv/backup/data/ansible` :

```bash
[web]
srvweb ansible_host=192.168.10.20 ansible_user=gabriel

[db]
srvdb ansible_host=192.168.10.30 ansible_user=gabriel

[backup]
vmbackup ansible_host=192.168.10.40 ansible_user=gabriel

[all:vars]
ansible_become=true
ansible_become_method=sudo
```

Crée all.yml dans `/srv/backup/data/ansible/group_vars` :

```bash
backup_ip: "192.168.10.40"

backup_user: "backup"
backup_root_dir: "/backup"
backup_web_dir: "/backup/srvweb"
backup_db_dir: "/backup/srvdb"

restic_project_dir: "/srv/backup"
restic_repo_dir: "/srv/backup/data"
restic_config_dir: "/srv/backup/config"
restic_logs_dir: "/srv/backup/logs"
restic_password: "changeme"
```

Playbook principal dans `/srv/backup/data/ansible` :

site.yml

```bash
---
- name: Configuration commune à toutes les VMs
  hosts: all
  roles:
    - common

- name: Configuration de la VM Backup
  hosts: backup
  roles:
    - vmbackup

- name: Configuration du serveur Web
  hosts: web
  roles:
    - srvweb

- name: Configuration du serveur Base de Données
  hosts: db
  roles:
    - srvdb
```

Créer les main.yml dans les roles(common, srvweb, srvdb, vmbackup) dans `/srv/backup/data/ansible/roles` :

1. Dans `/srv/backup/data/ansible/roles/common/tasks` :

main.yml

```bash
---
- name: Update apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Install base packages
  apt:
    name:
      - openssh-server
      - rsync
      - ca-certificates
      - curl
      - gnupg
    state: present
```

2. Dans `/srv/backup/data/ansible/roles/srvdb/tasks` :

main.yml

```bash
---
- name: Deploy db backup script
  copy:
    dest: /usr/local/bin/backup_db.sh
    mode: "0755"
    content: |
      #!/bin/bash
      echo "Backup lancé à $(date)" >> /var/log/backup_db.log
      mysqldump -u root -p'TonMotDePasse' infra_db > /tmp/infra_db.sql
      rsync -avz /tmp/infra_db.sql {{ backup_user }}@{{ backup_ip }}:{{ backup_db_dir }}/
      rm /tmp/infra_db.sql

- name: Ensure root has SSH keypair (db)
  openssh_keypair:
    path: /root/.ssh/id_ed25519
    type: ed25519
    comment: "backup-db"
  register: db_key

- name: Authorize SRVDB key on VMBACKUP for backup user
  authorized_key:
    user: "{{ backup_user }}"
    key: "{{ db_key.public_key }}"
  delegate_to: vmbackup

- name: Install cron for db backup (daily 03:00)
  cron:
    name: "Backup DB to VMBACKUP (daily)"
    minute: "0"
    hour: "3"
    job: "/usr/local/bin/backup_db.sh"
```

3. Dans `/srv/backup/data/ansible/roles/srvweb/tasks` :


```bash
---
- name: Deploy web backup script
  copy:
    dest: /usr/local/bin/backup_web.sh
    mode: "0755"
    content: |
      #!/bin/bash
      echo "Backup web lancé à $(date)" >> /var/log/backup_web.log
      rsync -avz /var/www/html/ {{ backup_user }}@{{ backup_ip }}:{{ backup_web_dir }}/

- name: Ensure root has SSH keypair (web)
  openssh_keypair:
    path: /root/.ssh/id_ed25519
    type: ed25519
    comment: "backup-web"
  register: web_key

- name: Authorize SRVWEB key on VMBACKUP for backup user
  authorized_key:
    user: "{{ backup_user }}"
    key: "{{ web_key.public_key }}"
  delegate_to: vmbackup

- name: Install cron for web backup (daily 02:00)
  cron:
    name: "Backup WEB to VMBACKUP (daily)"
    minute: "0"
    hour: "2"
    job: "/usr/local/bin/backup_web.sh"
```

4. Dans `/srv/backup/data/ansible/roles/vmbackup/tasks` :

main.yml

```bash                                                   
---
- name: Install packages needed on backup VM
  apt:
    name:
      - cron
      - docker-compose-plugin
    state: present

- name: Create backup directories
  file:
    path: "{{ item }}"
    state: directory
    mode: "0700"
  loop:
    - "{{ backup_root_dir }}"
    - "{{ backup_web_dir }}"
    - "{{ backup_db_dir }}"

- name: Ensure backup user exists
  user:
    name: "{{ backup_user }}"
    shell: /bin/bash
    create_home: yes

- name: Ensure SSH dir for backup user
  file:
    path: "/home/{{ backup_user }}/.ssh"
    state: directory
    owner: "{{ backup_user }}"
    group: "{{ backup_user }}"
    mode: "0700"

- name: Install Docker (get.docker.com) if not present
  shell: "curl -fsSL https://get.docker.com | sh"

  args:
    creates: /usr/bin/docker

- name: Create Restic project folders
  file:
    path: "{{ item }}"
    state: directory
    mode: "0755"
  loop:
    - "{{ restic_project_dir }}"
    - "{{ restic_repo_dir }}"
    - "{{ restic_config_dir }}"
    - "{{ restic_logs_dir }}"

- name: Write restic password file
  copy:
    dest: "{{ restic_config_dir }}/restic.pass"
    content: "{{ restic_password }}\n"
    mode: "0600"

- name: Deploy docker-compose.yml for restic
  copy:
    dest: "{{ restic_project_dir }}/docker-compose.yml"
    mode: "0644"
    content: |
      services:
        restic:
          image: restic/restic:latest
          volumes:
            - {{ restic_repo_dir }}:/backup
            - {{ backup_web_dir }}:/data-srvweb:ro
            - {{ backup_db_dir }}:/data-srvdb:ro
            - {{ restic_config_dir }}:/config:ro
          environment:

            RESTIC_REPOSITORY: /backup
            RESTIC_PASSWORD_FILE: /config/restic.pass

- name: Deploy backup script (Restic)
  copy:
    dest: "{{ restic_config_dir }}/backup.sh"
    mode: "0755"
    content: |
      #!/bin/bash
      set -e
      DATE=$(date +"%Y-%m-%d_%H-%M")
      LOG_DIR="{{ restic_logs_dir }}"
      LOG_FILE="$LOG_DIR/backup_$DATE.log"
      mkdir -p "$LOG_DIR"
      cd "{{ restic_project_dir }}"
      echo "=== Backup started at $(date) ===" >> "$LOG_FILE"
      /usr/bin/docker compose run --rm restic backup /data-srvweb /data-srvdb >> "$LOG_FILE" 2>&1
      /usr/bin/docker compose run --rm restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune >> "$LOG_FILE" 2>&1
      echo "=== Backup finished at $(date) ===" >> "$LOG_FILE"

- name: Initialize restic repository if needed
  shell: "/usr/bin/docker compose run --rm restic snapshots >/dev/null 2>&1 || /usr/bin/docker compose run --rm restic init"
  args:
    chdir: "{{ restic_project_dir }}"

- name: Install cron for restic backup (daily 02:00)
  cron:
    name: "Restic backup (daily)"
    minute: "0"
    hour: "2"
    job: "{{ restic_config_dir }}/backup.sh"
```

## Créer le Playbook :


Crée le site.yml dans `/srv/backup/data/ansible` : 

```bash
---
- name: Configuration commune à toutes les VMs
  hosts: all
  roles:
    - common

- name: Configuration de la VM Backup
  hosts: backup
  roles:
    - vmbackup

- name: Configuration du serveur Web
  hosts: web
  roles:
    - srvweb

- name: Configuration du serveur Base de Données
  hosts: db
  roles:
    - srvdb
```


## Conteneurisation (Restic)

Restic est exécuté uniquement sur la VM BACKUP via Docker Compose.

docker-compose.yml

```bash
services:
  restic:
    image: restic/restic:latest
    volumes:
      - /srv/backup/data:/backup
      - /backup/srvweb:/data-srvweb:ro
      - /backup/srvdb:/data-srvdb:ro
      - /srv/backup/config:/config:ro
    environment:
      RESTIC_REPOSITORY: /backup
      RESTIC_PASSWORD_FILE: /config/restic.pass
```

## Automatisation


Crontab 

```bash
crontab -e
```

```bash
#Ansible: Restic backup (daily)
0 2 * * * /srv/backup/config/backup.sh
```

## Sécurité

- Accès SSH par clé uniquement

- Utilisateur dédié backup

- Permissions restrictives (0700 / 0600)

- Mot de passe Restic stocké dans un fichier protégé

## Exécution

```bash
/srv/backup/config/backup.sh
```

## Vérification

```bash
cd /srv/backup
docker compose run --rm restic snapshots
```

```bash
docker compose run --rm restic check
```

Test de restauration :

```bash
docker compose run --rm restic restore latest --target /backup/restore-test
```

# Supervision

## Installer Node Exporter sur les 3 VMs (SRVWEB/SRVDB/VMBACKUP)

```bash
sudo apt update
sudo apt install -y prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter
sudo systemctl status prometheus-node-exporter --no-pager
```

Test sur chaque vm si ça s'affiche : 

```bash
root@VMSERVDATA:/home/gabriel# curl http://localhost:9100/metrics | head
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0# HELP apt_autoremove_pending Apt packages pending autoremoval.
# TYPE apt_autoremove_pending gauge
apt_autoremove_pending 0
# HELP apt_package_cache_timestamp_seconds Apt update last run time.
# TYPE apt_package_cache_timestamp_seconds gauge
apt_package_cache_timestamp_seconds 1.769288877203626e+09
# HELP apt_upgrades_pending Apt packages pending updates by origin
# TYPE apt_upgrades_pending gauge
apt_upgrades_pending{arch="all",origin="Debian:trixie-security/stable-security"} 4
apt_upgrades_pending{arch="all",origin="Debian:trixie/stable"} 23
100 56988    0 56988    0     0   120k      0 --:--:-- --:--:-- --:--:--  120k
curl: (23) Failure writing output to destination, passed 633 returned 356
root@VMSERVDATA:/home/gabriel#
```

## Déployer Prometheus + Grafana sur VMBACKUP (Docker Compose)

```bash
mkdir -p /srv/monitoring
cd /srv/monitoring
mkdir -p prometheus
```

### Prometheus config(toujours sur vmBackup)

```bash
nano /srv/monitoring/prometheus/prometheus.yml
```


```bash
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'nodes'
    static_configs:
      - targets:
          - '192.168.10.40:9100'  # VMBACKUP
          - '192.168.10.20:9100'  # SRVWEB
          - '192.168.10.30:9100'  # SRVDB
```


### docker-compose monitoring

On crée /srv/monitoring/docker-compose.yml :


```bash
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: unless-stopped

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    restart: unless-stopped
```

On lance enfin : 

```bash
docker compose up -d
docker compose ps
```

### Sur Prometheus

Taper dans un moteur de recherche :

```bash
http://192.168.56.116:9090/
```

Cliquer sur Targets et vous voyez si les machines sont bien UP.

### Ajouter Prometheus dans Grafana

Dans Grafana :

Connections / Data sources
Add data source → Prometheus
URL : http://prometheus:9090
Save & test

Vous serez redirigé sur le dashboard.
