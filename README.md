### INFRA 

# Architecture et configuration du réseau 

## Réseau sur 10.6.1.10, en 4 machines :


- ServeurDnsNAT = Rocky linux


CONFIG


Créer l'ip statique :


```
nano /etc/sysconfig/network-scripts/ifcfg-enp0s9
```



```
[root@serveur gabriel]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:a8:fe:a0 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
       valid_lft 85801sec preferred_lft 85801sec
    inet6 fd17:625c:f037:2:a00:27ff:fea8:fea0/64 scope global dynamic noprefixroute
       valid_lft 86155sec preferred_lft 14155sec
    inet6 fe80::a00:27ff:fea8:fea0/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:61:cd:3b brd ff:ff:ff:ff:ff:ff
    inet 10.6.1.14/24 brd 10.6.1.255 scope global noprefixroute enp0s9
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe61:cd3b/64 scope link
       valid_lft forever preferred_lft forever
```


Configurer le script pour la sauvegarde du site web : 

```
#!/bin/bash
# Script de sauvegarde ServeurWeb -> Rocky
# Auteur : Gabriel Cochet
# Objectif : Sauvegarde toutes les heures avec rotation

# Variables
SRC="gabriel@10.6.1.11:/var/www/"
DEST="/srv/backups/ServeurWeb"
LOG="/var/log/backup-ServeurWeb.log"
DATE=$(date +%Y-%m-%d_%H-%M)

# Création du répertoire si inexistant
mkdir -p $DEST

# Sauvegarde avec rsync
rsync -avz --delete \
    --exclude='*.log' \
    --exclude='cache/' \
    -e "ssh -i /root/.ssh/id_ed25519_nopass" \
    $SRC $DEST/$DATE >> $LOG 2>&1

# Vérification du succès
if [ $? -eq 0 ]; then
    echo "Sauvegarde terminée à $DATE" >> $LOG
else
    echo "Erreur de sauvegarde à $DATE" >> $LOG
fi

# Rotation : garder les 24 dernières sauvegardes
cd $DEST
ls -1t | tail -n +25 | xargs rm -rf
```


Attribuer les droits au script sauvegarde servWeb :

```
chmod 700 /usr/local/bin/backup-ServeurWeb.sh
```

Mettre en place le script dans le crontab :

```
crontab -e
```

```
*/5 * * * * /usr/local/bin/backup-ServeurWeb.sh
```


Vérifier les logs de sauvegardes du site web ..


```
cat /var/log/backup-ServeurWeb.log
```




- Client = Debian  



CONFIG


```
root@CLIENT1:/home/gabriel# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:b7:11:fc brd ff:ff:ff:ff:ff:ff
    altname enx080027b711fc
    inet 10.6.1.15/24 brd 10.6.1.255 scope global enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feb7:11fc/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever
```


- ServeurWeb = debian 


CONFIG


```
root@CLIENT2:/home/gabriel# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:0a:2d:1d brd ff:ff:ff:ff:ff:ff
    altname enx0800270a2d1d
    inet 10.6.1.11/24 brd 10.6.1.255 scope global enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe0a:2d1d/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever
```


- VM NAS = Rocky linux


CONFIG


```
[root@NAS gabriel]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:ac:8b:b0 brd ff:ff:ff:ff:ff:ff
    inet 10.6.1.17/24 brd 10.6.1.255 scope global noprefixroute enp0s9
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feac:8bb0/64 scope link
       valid_lft forever preferred_lft forever
```



## Setup du serveur DNS sur 




```

```



## Setup du serveur Web 



