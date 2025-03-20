### INFRA 


```
projet : Squid Server
```

#### ROUTEUR/SERVEUR SQUID

```
-rocky linux
-IP privée
-DNS
-Accès Internet
```

Interface réseau

```
[root@localhost gabrielcochet]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:20:f3:17 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
       valid_lft 520sec preferred_lft 520sec
    inet6 fe80::a00:27ff:fe20:f317/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:42:4b:8a brd ff:ff:ff:ff:ff:ff
    inet 10.7.1.254/24 brd 10.7.1.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe42:4b8a/64 scope link
       valid_lft forever preferred_lft forever
```

#### Client  

```
-ubuntu debian
-IP privée
-Accès au adblock via vpn
-Connexion host-only via routeur
```

Interface réseau


```
gabriel@gabriel-VirtualBox:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:50:76:e1 brd ff:ff:ff:ff:ff:ff
    inet 10.7.1.103/24 brd 10.7.1.255 scope global noprefixroute enp0s3
       valid_lft forever preferred_lft forever
```


#### Installation de Squid sur le routeur :



Tout d'abord, installe Squid avec la commande :

```bash
sudo apt update && sudo apt install squid -y



systemctl status squid



sudo systemctl start squid



sudo systemctl enable squid

```

Édite le fichier de configuration principal de Squid :


```
sudo nano /etc/squid/squid.conf
```

Ajoute ou modifie les lignes suivantes :

```
# Définition des ACL (Access Control List)
acl localnet src 10.0.0.0/8  # Réseau local
acl localnet src 192.168.0.0/16  # Réseau local
acl client_ubuntu src 10.7.1.103  # Client spécifique
acl allowed_clients src 10.7.1.254  # Clients autorisés

# Liste des sites bloqués
acl blocked_sites dstdomain "/etc/squid/blocked_sites.txt"

# Définition des ports autorisés
acl SSL_ports port 443
acl Safe_ports port 80  # HTTP
acl Safe_ports port 443  # HTTPS
acl Safe_ports port 21   # FTP
acl Safe_ports port 1025-65535  # Ports non enregistrés

# Configuration du cache
cache_dir ufs /var/spool/squid 100 16 256

# Autoriser les clients spécifiques
http_access allow allowed_clients

# Bloquer les sites spécifiés dans le fichier blocked_sites.txt
http_access deny blocked_sites

# Sécurité : Bloquer les ports non sécurisés
http_access deny !Safe_ports

# Bloquer les connexions non SSL sur les ports sécurisés
http_access deny CONNECT !SSL_ports

# Autoriser la gestion depuis localhost
http_access allow localhost manager
http_access allow client_ubuntu
http_access deny manager

# Autoriser les clients du réseau local
http_access allow localnet
http_access allow localhost

# Bloquer tout le reste
http_access deny all

# Configuration du port d'écoute du proxy Squid
http_port 3128

# Logs et stockage
access_log /var/log/squid/access.log squid
cache_log /var/log/squid/cache.log
coredump_dir /var/spool/squid

# Rafraîchissement du cache
refresh_pattern ^ftp:           1440    20%     10080
refresh_pattern -i (/cgi-bin/|\?) 0     0%      0
refresh_pattern .               0       20%     4320
```

Créer le fichier contenant les sites interdits :


```
sudo nano /etc/squid/blocked_sites.txt
```

Ajoute les sites à bloquer :

```
facebook.com
youtube.com
twitter.com
```

Pour appliquer les changements, recharge la configuration de Squid :

```
sudo squid -k reconfigure
sudo systemctl restart squid
```


Vérifie que Squid écoute bien sur le port 3128 :


```
netstat -tulnp | grep squid
```

Teste une connexion en passant par le proxy :

```
curl -I -x http://10.7.1.254:3128 https://www.facebook.com
```
(bien évidemment changer l'ip selon votre machine)


Si Squid bloque correctement le site, la réponse devrait être :

```
HTTP/1.1 403 Forbidden
```

installer logorotate : 

```
sudo apt install logrotate -y   # (Debian/Ubuntu)
sudo yum install logrotate -y   # (CentOS/RHEL)
```

Configurer logrotate pour Squid
Crée ou modifie le fichier de configuration spécifique à Squid :

```
sudo nano /etc/logrotate.d/squid
```

Ajoute le contenu suivant :

```
plaintext
Copy
Edit
/var/log/squid/*.log {
    daily                  # Rotation quotidienne
    rotate 7               # Conserver les 7 derniers fichiers
    compress               # Compresser les anciens logs (gzip)
    delaycompress          # Ne pas compresser le log le plus récent
    missingok              # Ne pas afficher d'erreur si le fichier n'existe pas
    notifempty             # Ne pas faire la rotation si le log est vide
    create 640 proxy adm   # Créer un nouveau fichier log après rotation
    postrotate
        /usr/sbin/squid -k rotate
    endscript
}
```

Vérifie si logrotate fonctionne correctement avec :

```
sudo logrotate -d /etc/logrotate.d/squid
```

Si Squid ne fonctionne pas comme prévu, consulte les logs :

```
tail -f /var/log/squid/access.log
tail -f /var/log/squid/cache.log
```