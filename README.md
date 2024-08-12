
# Topologie du laboratoire 

```none
  DEVICE NAME        IPv4 ADDRESS              IPv6 ADDRESS
+--------------+-----------------------+-----------------------------+
| grpX-cli     | 100.100.X.2 (eth0)    | fd89:59e0:X::2 (eth0)       |
+--------------+-----------------------+-----------------------------+
| grpX-ns1     | 100.100.X.130 (eth0)  | fd89:59e0:X:128::130 (eth0) |
+--------------+-----------------------+-----------------------------+
| grpX-ns2     | 100.100.X.131 (eth0)  | fd89:59e0:X:128::131 (eth0) |
+--------------+-----------------------+-----------------------------+
| grpX-resolv1 | 100.100.X.67 (eth0)   | fd89:59e0:X:64::67 (eth0)   |
+--------------+-----------------------+-----------------------------+
| grpX-resolv2 | 100.100.X.68 (eth0)   | fd89:59e0:X:64::68 (eth0)   |
+--------------+-----------------------+-----------------------------+
| grpX-rtr     | 100.64.1.X (eth0)     | fd89:59e0:X::1 (eth1)       |
|              | 100.100.X.65 (eth2)   | fd89:59e0:X:64::1 (eth2)    |
|              | 100.100.X.193 (eth4)  | fd89:59e0:X:192::1 (eth4)   |
|              | 100.100.X.129 (eth3)  | fd89:59e0:X:128::1 (eth3)   |
|              | 100.100.X.1 (eth1)    | fd89:59e0:0:1::X (eth0)     |
+--------------+-----------------------+-----------------------------+
| grpX-soa     | 100.100.X.66 (eth0)   | fd89:59e0:X:64::66 (eth0)   |
+--------------+-----------------------+-----------------------------+
```
Durant cette pratique, nous n'accéderons qu'aux équipements suivants :

-   **grpX-cli** : client
-   **grpX-soa** : serveurs faisant autorité cachés (primaires)
-   **grpX-ns1** et **grpX-ns2** : serveurs secondaires faisant autorité

**NOTE:** Dans tout ce labo, veillez à toujours remplacer **_X_** par votre numéro de Groupe dans les adresses IP, le nom du serveur et tout autre endroit où cela est nécessaire. De même pour < _lab_domain_ > à remplacer par le nom de domaine enregistré pour le cours.

# Configurer le serveur principal faisant autorité (BIND)

## Introduction

Nous allons configurer un serveur faisant autorité caché et créer la zone faisant autorité _grpX_ .< _lab_domain_ >.te-labs.training.

## Ce que nous savons déjà

Notre parent (< _lab_domain_ >.te-labs.training) a déjà créé ce qui suit dans sa propre zone :

```shell
; grpX
grpX             NS          ns1.grpX.<lab_domain>.te-labs.training.
grpX             NS          ns2.grpX.<lab_domain>.te-labs.training.
; ---Placeholder for grpX DS record (DO NOT MANUALLY EDIT THIS LINE)---
ns1.grpX         A           100.100.X.130
ns1.grpX         AAAA        fd89:59e0:X:128::130
ns2.grpX         A           100.100.X.131
ns2.grpX         AAAA        fd89:59e0:X:128::131

```

Notre configuration de zone doit être compatible avec cela.

## Définition de la zone faisant autorité

Nous utilisons le conteneur SOA (hidden primary authoritative) [ **grpX-soa** ]

Nous allons dans le `/etc/bind`répertoire et clonons le fichier db.empty dans un nouveau répertoire que nous créons pour nos zones :

```none
$ sudo mkdir -p /etc/bind/zones
$ cp db.empty zones/db.grpX
```

Le contenu de la zone doit être au minimum :

```none
; grpX 

$TTL    300
@       IN      SOA     grpX.<lab_domain>.te-labs.training. dnsadmin.<lab_domain>.te-labs.training. (                                            
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;

; grpX 
@             NS           ns1.grpX.<lab_domain>.te-labs.training.
@             NS           ns2.grpX.<lab_domain>.te-labs.training.

ns1         A           100.100.X.130
ns1         AAAA        fd89:59e0:X:128::130
ns2         A           100.100.X.131
ns2         AAAA        fd89:59e0:X:128::131
```

> Vous pouvez ajouter plus d'enregistrements selon vos envies.

Une fois terminé, il est important de vérifier le format du fichier de zone. Pour ce faire, utilisez la commande **_named-checkzone avec les paramètres appropriés._**

Dans le fichier de configuration **_/etc/bind/named.conf.local_** nous mettons la déclaration zone :

```none
zone "grpX.<lab_domain>.te-labs.training" {                                                                               
        type master;                                                                                                  
        file "/etc/bind/zones/db.grpX";                                                                                     
        allow-transfer { any; };                                                                                      
}; 
```

Nous redémarrons le serveur et vérifions :

```none
rndc reload
```

```none
root@soa:/etc/bind# dig @localhost soa grpX.<lab_domain>.te-labs.training.                                                

; <<>> DiG 9.16.1-Ubuntu <<>> @localhost soa grpX.<lab_domain>.te-labs.training.
; (2 servers found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64339
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 270e2c46ed443c1c01000000609c59f04ba85015ff71998d (good)
;; QUESTION SECTION:
;grpX.<lab_domain>.te-labs.training.        IN      SOA

;; ANSWER SECTION:
grpX.<lab_domain>.te-labs.training. 30 IN   SOA     grpX.<lab_domain>.te-labs.training. dnsadmin.<lab_domain>.te-labs.training. 1 604800 86400 2419200 86400

;; Query time: 0 msec
;; SERVER: ::1#53(::1)
;; WHEN: Wed May 12 22:42:56 UTC 2021
;; MSG SIZE  rcvd: 170

```

# Configurer des serveurs secondaires faisant autorité

Ces serveurs sont ceux qui exposent notre zone publiquement (ils seront donc ouverts à tous les serveurs).

#### Nous configurons le premier serveur ns1 [ **ns1.grpX** ]

**Le serveur ns1 exécute BIND** (depuis ISC)

Nous allons dans le `/etc/bind`répertoire et créons un fichier qui contiendra notre fichier de zone dans le serveur de noms :

```none
$ sudo mkdir -p /etc/bind/zones
$ touch /etc/bind/zones/db.grpX.slave
```

Pour ce faire, dans le fichier **_/etc/bind/named.conf.local_** nous configurons les paramètres suivants :

```none
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "grpX.<lab_domain>.te-labs.training" {
        type slave;
        file "/etc/bind/zones/db.grpX.slave";
        masters { 100.100.X.66; };
};
```

Nous vérifions la configuration et s'il n'y a pas d'erreurs nous redémarrons le serveur :

```none
# named-checkconf
# systemctl restart bind9
```

Nous vérifions qu'il a redémarré correctement :

```none
# systemctl status bind9
```

```none
● named.service - BIND Domain Name Server
     Loaded: loaded (/lib/systemd/system/named.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/service.d
             └─lxc.conf
     Active: active (running) since Thu 2021-05-13 04:25:43 UTC; 9s ago
       Docs: man:named(8)
   Main PID: 739 (named)
      Tasks: 50 (limit: 152822)
     Memory: 103.9M
     CGroup: /system.slice/named.service
             └─739 /usr/sbin/named -f -u bind

May 13 04:25:43 ns1.grpX.<lab_domain>.te-labs.training named[739]: all zones loaded
May 13 04:25:43 ns1.grpX.<lab_domain>.te-labs.training named[739]: running
May 13 04:25:43 ns1.grpX.<lab_domain>.te-labs.training named[739]: zone grpX.<lab_domain>.te-labs.training/IN: Transfer started.
May 13 04:25:43 ns1.grpX.<lab_domain>.te-labs.training named[739]: transfer of 'grpX.<lab_domain>.te-labs.training/IN' from 100.100.2.66#53: connec>
May 13 04:25:43 ns1.grpX.<lab_domain>.te-labs.training named[739]: zone grpX.<lab_domain>.te-labs.training/IN: transferred serial 1
May 13 04:25:43 ns1.grpX.<lab_domain>.te-labs.training named[739]: transfer of 'grpX.<lab_domain>.te-labs.training/IN' from 100.100.2.66#53: Transf>
May 13 04:25:43 ns1.grpX.<lab_domain>.te-labs.training named[739]: transfer of 'grpX.<lab_domain>.te-labs.training/IN' from 100.100.2.66#53: Transf>
May 13 04:25:43 ns1.grpX.<lab_domain>.te-labs.training named[739]: zone grpX.<lab_domain>.te-labs.training/IN: sending notifies (serial 1)
May 13 04:25:43 ns1.grpX.<lab_domain>.te-labs.training named[739]: managed-keys-zone: Key 20326 for zone . is now trusted (acceptance timer com>
May 13 04:25:43 ns1.grpX.<lab_domain>.te-labs.training named[739]: resolver priming query complete
```

#### Nous configurons maintenant le serveur ns2 [ **ns2.grpX** ]

**Le serveur ns2 exécute NSD** (de NLnet Labs)

Pour ce faire, dans le fichier **_/etc/nsd/nsd.conf_** nous configurons les paramètres suivants :

```none
# NSD configuration file for Debian.
#
# See the nsd.conf(5) man page.
#
# See /usr/share/doc/nsd/examples/nsd.conf for a commented
# reference config file.
#
# The following line includes additional configuration files from the
# /etc/nsd/nsd.conf.d directory.

include: "/etc/nsd/nsd.conf.d/*.conf"

server:
    zonesdir: "/etc/nsd"

pattern:
    name: "fromprimary"
    allow-notify: 100.100.X.66 NOKEY
    request-xfr: AXFR 100.100.X.66 NOKEY

zone:
    name: "grpX.<lab_domain>.te-labs.training"
    zonefile: "grpX.<lab_domain>.te-labs.training.forward"
    include-pattern: "fromprimary"
```

Nous vérifions la configuration et s'il n'y a pas d'erreurs, redémarrons le serveur :

```none
# nsd-checkconf /etc/nsd/nsd.conf
# systemctl restart nsd
```

Nous vérifions qu'il a redémarré correctement :

```none
# systemctl status nsd
```

```none
● nsd.service - Name Server Daemon
     Loaded: loaded (/lib/systemd/system/nsd.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/service.d
             └─lxc.conf
     Active: active (running) since Thu 2021-05-13 05:02:35 UTC; 1min 22s ago
       Docs: man:nsd(8)
   Main PID: 638 (nsd)
      Tasks: 3 (limit: 152822)
     Memory: 114.5M
     CGroup: /system.slice/nsd.service
             ├─638 /usr/sbin/nsd -d
             ├─639 /usr/sbin/nsd -d
             └─640 /usr/sbin/nsd -d

May 13 05:02:35 ns2.grpX.<lab_domain>.te-labs.training systemd[1]: Starting Name Server Daemon...
May 13 05:02:35 ns2.grpX.<lab_domain>.te-labs.training nsd[638]: nsd starting (NSD 4.1.26)
May 13 05:02:35 ns2.grpX.<lab_domain>.te-labs.training nsd[638]: [2021-05-13 05:02:35.865] nsd[638]: notice: nsd starting (NSD 4.1.26)
May 13 05:02:35 ns2.grpX.<lab_domain>.te-labs.training nsd[639]: nsd started (NSD 4.1.26), pid 638
May 13 05:02:35 ns2.grpX.<lab_domain>.te-labs.training nsd[639]: [2021-05-13 05:02:35.922] nsd[639]: notice: nsd started (NSD 4.1.26), pid 638
May 13 05:02:35 ns2.grpX.<lab_domain>.te-labs.training systemd[1]: Started Name Server Daemon.
```

## Testez la configuration et la propagation de votre zone.

#### Utilisez l'outil de fouille pour tester le domaine

Nous allons maintenant utiliser l'outil _dig_ pour vérifier notre propre configuration de zone et propagation, puis faire de même pour un ou deux autres groupes de la classe et partager des commentaires. Depuis votre client, exécutez les requêtes dig suivantes. Toutes doivent renvoyer une réponse, sinon vous devez revoir vos configurations avant de continuer :

1.  creuser soa _grpX_ .< _domaine_laboratoire_ >.te-labs.training. @100.100.X.66
2.  creuser soa _grpX_ .< _domaine_laboratoire_ >.te-labs.training. @100.100.X.130
3.  creuser soa _grpX_ .< _domaine_laboratoire_ >.te-labs.training. @100.100.X.131
4.  creuser soa _grpX_ .< _domaine_laboratoire_ >.te-labs.training. @100.100.X.131 +short
5.  creuser soa _grpX_ .< _domaine_laboratoire_ >.te-labs.training. @100.100.X.131 +multi
6.  creuser NS _grpX_ .< _domaine_laboratoire_ >.te-labs.training. @100.100.X.130
7.  creuser NS _grpX_ .< _domaine_laboratoire_ >.te-labs.training. @100.100.X.130 +multi

#### Transfert de zone

Nous essayons maintenant d'effectuer un transfert de zone manuel en utilisant ns1, ns2 et l'adresse IP du serveur SOA à partir de notre machine cliente. Que remarquez-vous ?

Comment pouvons-nous protéger notre fichier de zone pour empêcher les gens d'accéder à l'intégralité de son contenu ?
