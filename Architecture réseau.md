**Travail à réaliser**

Ci-dessous un schéma de l’organisation du réseau que vous aurez à mettre en place. Vous choisirez vos noms de machines qui remplaceront ROUTEUR, INTERNE et EXTERNE.

![](Aspose.Words.3043882c-0868-4b50-a159-dc74a3c85fbb.001.png)



Interne : 10.0.13.13/24 dev jaune

Routeur : 
* ip réseau interne -> 10.0.13.254/24 dev jaune 
* ip réseau externe -> 192.168.13.254/24 dev jaune

Externe : 192.168.13.1/24 dev jaune


## Mettre en place la vlan :

Avant de commencer, nous avons mis en place le VLAN, qui permettrait, plus tard, de donner une configuration réseau au client externe grace au serveur DHCP de routeur.

Nous avons commencé par charger le module noyau nécessaire en utilisant la commande suivante :
```
modprobe 8021q
```
Cela a permis d'activer le support VLAN dans le noyau

Ensuite, nous avons créé une interface virtuelle VLAN appelée jaune.13 sur la base de l'interface physique jaune. Cette opération a été effectuée avec la commande :
```
ip link add link jaune name jaune.13 type vlan id 13
```
Pour activer la nouvelle interface virtuelle VLAN, nous avons exécuté la commande suivante :
```
ip link set jaune.13 up
```
Cela a permis de mettre en service l'interface virtuelle VLAN jaune.13.

Chacune des machines connectées au VLAN a ensuite reçu une adresse IP manuellement. 

Pour le routeur, nous avons utilisé la commande :

```
ip a add 10.0.13.254/4 dev jaune.13
```
Et pour le client :
```
ip a add 10.0.13.13/4 dev jaune.13
```
Nous pouvons observer que lorsque nous effectuons ```ip a ``` : 


## Pour la partie DHCP :

En tout premier nous devons activer le forwarding : 

Il faut en premier modifier le fichier de conf. :
```
sudo nano /etc/sysctl.conf
```

Nous avons ensuite décommenter la ligne suivante, qui sert à activer l'ip forwarding :
```
#net.ipv4.ip_forward=1
```


Puis nous avons recharger le fichier de conf. :
```
sysctl -p /etc/sysctl.conf
```

Le résultat ressemble à cela :
```bash 
root@debian:/home/tdreseau# sysctl -p /etc/sysctl.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.all.autoconf = 0
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.default.autoconf = 0
```

Avec la commande ```sudo nano /etc/dhcp/dhcpd.conf```, à la fin du fichier, nous rajoutons :
```bash
subnet 10.0.13.0 netmask 255.255.255.0 {
  range 10.0.13.1 10.0.13.253;
  option domain-name-servers 10.0.13.254;
  option domain-name "serveur_dns13.com";
  option routers 10.0.13.254;
  host CLIENT {
      hardware ethernet 04:8d:38:cf:7e:8a;
      fixed-address 10.0.13.13;
   }
 }
```
Dans ```/etc/default/isc–dhcp-server``` nous rajoutons l'interface avec laquel nous voudrons recuperer notre configuration réseau, dans notre cas le VLAN donc jaune.13 :
```
sudo nano /etc/default/isc-dhcp-server
```
Dans le fichier on modifie :
```
. . .
interfacesv4="jaune.13"
```

Nous redémarrons le service :
```
sudo systemctl restart isc-dhcp-server
```
Nous avons installé et configuré un serveur DHCP (paquet isc-dhcp-server). Celui-ci nous fournira une route par défaut vers la machine ROUTEUR, le nom de domaine, et l’IP du serveur DNS. L'hôte INTERNE doit se voir délivrer une adresse fixe en fonction de son adresse MAC.

Comme nous pouvons l'observez :
...

## Pour la partie DNS :
Le DNS (Domain Name System) est un système qui traduit les noms de domaine en adresses IP, facilitant la navigation sur Internet en utilisant des noms conviviaux au lieu de mémoriser des adresses numériques.

Dans /etc/bind\_sae\_dns13.db on crée un nouveau fichier avec :
```
/etc/bind/sae_dns13.db
```
On rajoute les lignes suivantes :
```bash
$ttl 38400
@       IN      SOA     ns13.serveur_dns13.com. postmaster.ns13.sae_dns13.com. (
                1       ; Numero de serie, a incrementer a chaque modification
                10800   ; Rafraichissement
                3600    ; Nouvel essai apres 1 heure
                604800  ; Obsolescence apres 1 semaine
                86400 ) ; TTL minimale de 1 jour

              IN    NS    ns13.serveur_dns13.com.
ns13       IN    A     192.168.13.254
www      IN    CNAME  ns13.serveur_dns13.com.
routeur   IN    CNAME  ns13.serveur_dns13.com.


```

* La première section (@ IN SOA...) est l'enregistrement de la zone qui spécifie les informations sur le domaine.
* IN NS ns13.sae_dns13.com -> déclare le serveur de noms principal pour la zone.
* ns13 IN A 10.0.13.254 -> spécifie l'adresse IP du serveur.
* www IN CNAME interne.ns13.sae_dns13.com -> crée un alias (CNAME) pour le nom www, pointant vers interne.ns13.sae_dns13.com.
* interne IN A 10.0.13.13 et routeur IN A 10.0.13.254 -> définissent les adresses IP d'autres hôtes.

On redemarre le service BIND pour appliquer les modifications.:
```
sudo systemctl restart bind9
```
On test que ca fonctionne avec la commande suivante :
```
nslookup www.ns13.sae\_dns13.com
```
Et maintenant on modified named.conf avec :
```
sudo nano /etc/bind/named.conf
```
On modifie le fichier pour rajouter une nouvelle zone :
```bash
include "/etc/bind/named.conf.options"; include "/etc/bind/named.conf.local"; include "/etc/bind/named.conf.default-zones";

zone "serveur_dns13.com" in { // déclaration de la zone

  type master; // déclaration type maître
  file "/var/cache/bind/sae_dns13.db";// on indique le fichier contena$ };
```
```
nano /etc/resolv.conf
```

## Pour la partie HTTP :

Créer un fichier de configuration : ``` nano /etc/apache2/site-available/monsite.conf ```
Dans le fichier de configuration monsite.conf :
```bash
<VirtualHost *:80>
  ServerAdmin webmaster@monsite.com
  DocumentRoot /var/www/html
  DirectoryIndex index.html
  <Directory /var/www/html>
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
  </Directory>
</VirtualHost>
```

Pour pouvoir activer ce site il faut faire la commande :
```bash
sudo a2ensite monsite.conf
```

Et enfin :
```bash
systemctl restart apache2
```
Ensuite nous allons télécharger notre MVC grâce à la commande suivante :
```bash
wget https://gitlab.univ-nantes.fr/pub/but/but2/r3.01/r3.01/-/archive/main/r3.01-main.zip?path=td/workspace
```

Nous allons dans le dossier et copions /app et /system dans /var/www/html :
```sudo cp -r \* /var/www/html```



