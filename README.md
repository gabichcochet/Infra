## PROJET INFRA

Architecture

🛣️ 1. VM-ROUTER
OS : Debian

Pourquoi Debian ?
-Très léger → parfait pour un routeur minimaliste
-Stabilité exemplaire → un routeur ne doit jamais planter
-Configuration réseau simple et standard
-Documentation immense pour : iptables, nftables, NAT, routage, DHCP, VPN
-Utilisé comme base dans de nombreuses appliances réseau (VyOS, IPFire, Untangle)

👉 C’est l’OS le plus logique pour un routeur Linux.

🌐 2. VM-SRV-DNS
OS : Debian

Pourquoi Debian ?
-Bind9 est historiquement mieux documenté sur Debian
-Très stable, idéal pour un service d’infrastructure
-Peu gourmand en ressources
-Maintenance simple, mises à jour non intrusives
-Parfait pour un DNS interne d’entreprise

👉 Debian = choix naturel pour les services réseau fondamentaux.

🌍 3. VM-SRV-WEB
OS : Debian

Pourquoi Debian ?
-Nginx/Apache sont extrêmement bien supportés
-Très léger → idéal pour une VM web
-Gestion simple des certificats HTTPS (Certbot)
-Compatible Docker si tu conteneurises ton service
-Très utilisé dans les environnements web professionnels

👉 Debian est le standard pour les serveurs web Linux.

🗄️ 4. VM-SRV-DATA (Base de données)
OS : Rocky Linux

Pourquoi Rocky Linux ?
-Compatible RHEL → standard des entreprises
-Très robuste pour les services critiques (DB, stockage)
-Cycle de support long (10 ans)
-Optimisé pour MariaDB/PostgreSQL
-Très utilisé dans les datacenters et environnements VMware/Proxmox

👉 Rocky = OS entreprise, parfait pour les bases de données.

💾 5. VM-BACKUP
OS : Debian

Pourquoi Debian ?

-Très léger
-Parfait pour des outils comme Borg, Restic, Rsync
-Maintenance simple
-Idéal si tu veux un serveur de backup minimaliste

# 🖧 VM Routeur — Configuration Complète

## 🎯 Rôle de la VM Routeur
La VM Routeur assure la communication entre les différentes zones du réseau et Internet.  
Elle remplit quatre fonctions principales :

- Fournir l’accès Internet au LAN et à la DMZ via **NAT**
- Assurer le **routage** entre les réseaux internes
- Appliquer un **pare‑feu** pour contrôler les flux
- Isoler la **DMZ** du **LAN** pour des raisons de sécurité

---

## 🌐 Configuration réseau

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
root@debian:/home/gabriel# systemctl restart networking
```

### Activation du routage IP dans `/etc/sysctl.conf`

```bash
net.ipv4.ip_forward = 1
```

### Application 

```bash
root@debian:/home/gabriel# sudo sysctl -p
```

## Configuration IPTABLES

### Configuration NAT

```bash
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```

### Pare-feu iptables

```bash
root@debian:/home/gabriel# nano /etc/iptables/rules.v4
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
root@debian:/home/gabriel# sudo netfilter-persistent save
```

```bash
root@debian:/home/gabriel# sudo netfilter-persistent reload
```

### Vérification totale

```bash
root@debian:/home/gabriel# iptables -t nat -L -v -n
```

```bash 
root@debian:/home/gabriel# systemctl status netfilter-persistent
```

