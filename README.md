# Ansible Raspberry Pi Box

Projet pour créer avec un raspberry une box qui prend en entrée un iphone et en sortie 1 reseau local et 2 wifi  
Projet pour surtout sauvegarder ce code et la methode pour de faire
Je n'ai pas pour volonté de faire quelque chose qui marche partout, mais qui marche assez bien pour chez moi

## WIP :

"Oh mince ! Plus de forfait ! Je dois connecter le raspberry a un wifi plutôt qu'au telephone"  
todo: README

Connection En wifi
```
ansible-playbook --vault-password-file pass.txt -i inventory raspberry.yml
```

Connection En Telephone
```
ansible-playbook --vault-password-file pass.txt -i inventory raspberry.yml --skip-tags config-srcwifi
```

### Status

A verifier :

- la stabilité de l'adresse mac de l'iphone en partage de connection USB

A faire :

- Relire tout pour assurer le sens du README

### Pour ceux qui ons deja lu le reste

Toute les operations sont faisables à la suite depuis le status d'access rapide au raspberry  
Vous pouvez parfaitement juste ecrire l'image sur la carte SD, faire `Installation du raspberry`, laisser votre telephone en point d'access wifi et faire tourner le script  
En suivant les quelques petite instruction en dessous, cela marche tout autant  
⚠️ **Attention tout de meme à bien remplir l'inventory et le .vault sans erreur** ⚠️  
⚠️ **Cet operation est longue prévoyez 1 Heure** ⚠️  
L'operation la plus longue est l'etape d'installation dans ansible

Pour tout exécuter :

```
ansible-playbook --vault-password-file pass.txt -i inventory raspberry.yml
```

## Réseau et environment

J'ai a ma disposition

- un Raspberry Pi 3B+
- une carte SD de 16 Go Classe 10
- un écran
- un clavier
- une souris
- un iphone 5S avec un forfait 4G pour me fournir un hotspot wifi / partage internet en USB
- un réseau local sans dhcp ni gateway (un switch tout seul on peut dire, pas de source, pas de box)
- une carte wifi en usb à onde porteuse que j'ai achetée il y a longtemps (2.4Ghz)
- une clef usb wifi Archer T3U
- un ordinateur qui exécutera l'ansible, connecté au reseau ET a une autre source d'internet (wifi par exemple)

J'ai plusieurs configurations pour pouvoir manipuler mon raspberry sur la carte réseau de l'ordinateur

Config "IP fixe" : cela vous servira à être present sur le reseau et internet quoi qu'il arrive  
Je vous conseil de vous configurer ainsi pour être sur du fonctionnement  
À définir manuellement  
Adresse IP : 192.168.2.30  
Sous-réseau : 255.255.255.0  
Routeur : (vide)  
Server DNS : (vide)

Config "Automatique" : cela vous servira à tester si tout fonctionne  
Définir en configuration DHCP

Sur l'ordinateur, pensez à installer ansible  
Mon ordinateur :

```
$ ansible --version
ansible [core 2.12.4]
  (...)
```

## Installation du raspberry

J'utilise une image de raspberry daté de 2019-04-08  
"Raspbian Stretch Full"  
Le fichier s'appelle "2019-04-08-raspbian-stretch-full.img"  
Copiez l'image sur la carte SD du raspberry

Branchez le raspberry a l'écran, au clavier, a la souris et au réseau  
NE PAS BRANCHER PLUS, ou sinon udev vas être en capacité de définir les carte réseau d'un autre nom que ce que l'on a l'habitude et que l'on voudrait  
Définissez une IP fixe à votre raspberry et activez le ssh

```
$ nano /etc/network/interfaces.d/eth0
allow-hotplug eth0
iface eth0 inet static
  address 192.168.2.1
  netmask 255.255.255.0
  gateway 192.168.2.1
  dns-nameservers 4.4.4.4
  dns-nameservers 8.8.8.8

$ sudo systemctl enable ssh
$ sudo nano /etc/ssh/sshd_config
(...)
PubkeyAuthentication yes
(...)
AuthorizedKeysFile      /etc/ssh/authorized_keys
(...)
$ sudo systemctl start ssh
$ sudo systemctl enable ssh
```

À ce moment, il faut faire la première connection avec l'ordinateur  
Sur l'ordinateur définir une ip fixe puis faire un ssh sur le raspberry  
Pour moi j'ai un host et une config ssh

Pour définir l'IP fixe, je vous laisse faire, je ne connais pas votre système
J'ai décris les configurations reseau plus haut  

```
# commandes equivalentes
$ ssh rasp
$ ssh pi@192.168.2.1
```

À ce moment une verification de clef vous demande

```
The authenticity of host 'rasp (192.168.2.1)' can't be established.
ED25519 key fingerprint is SHA256:___________________________________________.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

À la place des "_" vous avez un hash que l'on peut retrouver depuis les fichiers du raspberry  
Notez bien le "ED25519", c'est important pour savoir quelle clef est utilisé

```
$ sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key
256 SHA256:___________________________________________ root@raspberrypi (ED25519)
```

Vérifiez que le hash est le même et sur l'ordinateur et dites "yes" dans ce cas  
À ce moment, vous devez entrer un mot de passe pour l'utilisateur "pi"  
Mot de passe par default : "raspberry"  
Pensez bien à le changer et de le noter dans un endroit sécurisé

Changer le mot de passe : 
```
# on verifie que l'on est bien "pi"
$ whoami
pi
# changer le mot de passe
$ passwd
```

Sur l'ordinateur, copiez votre clef publique  
Et collez la dans `/etc/ssh/authorized_keys`

```
$ cat ~/.ssh/id_rsa.pub
ssh-rsa ____..  ..____ atyklaxas@atyklaxas-ordinateur
```

```
$ nano /etc/ssh/authorized_keys
ssh-rsa ____..  ..____ atyklaxas@atyklaxas-ordinateur
```

Vous avez maintenant créé un access facile a votre raspberry  
Vous pouvez maintenant tester cela  
Sur l'ordinateur :

```
$ ssh rasp
Linux raspberrypi 5.15.32-v7+ #1538 SMP Thu Mar 31 19:38:48 BST 2022 armv7l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun May  8 19:20:43 2022 from 192.168.2.89
pi@raspberrypi:~ $
```

Au cas ou ansible failerais, je vous conseil de faire une image de la carte SD du raspberry a ce moment  
Je vous laisse choisir un logiciel pour faire vos images, moi j'utilise ubuntu avec le logiciel "Disk" sur un autre ordinateur

## Avant ansible

Pour que tout cela fonctionne, je vais vous indiquer les variables à remplir et fichier a créé

- `ip_rasp` IP du raspberry, normalement 192.168.2.1
- `wifi_ssid` SSID du wifi pour l'operation pre-connect, pour avoir internet durant la procedure
- `vault_wifi_password` Mot de passe du wifi a mettre dans le .vault
- `ap_wifi_ssid` Nom du SSID du raspberry, a noter que le reseau 5Ghz sera suivi de `-5Ghz`
- `vault_ap_wifi_password` Mot de passe wifi du point d'access
- `mac_eth0` Adresse mac de la carte Ethernet du raspberry *🔎1
- `mac_wlan0` Adresse mac de la carte Wifi du raspberry *🔎1
- `mac_eth1` Adresse mac de la carte "Ethernet" qu'est l'iphone *🔎2
- `mac_wlan1` Adresse mac de la carte Wifi 2.4Ghz *🔎3
- `mac_wlan2` Adresse mac de la carte Wifi 5Ghz *🔎3
- `fin_ip_eth0` Fin de l'adresse IP du raspberry sur le réseau eth0
- `fin_ip_wlan1` Fin de l'adresse IP du raspberry sur le réseau wlan1
- `fin_ip_wlan2` Fin de l'adresse IP du raspberry sur le réseau wlan2
- `eth0_subnet` Sous réseau du raspberry sur le réseau eth0
- `wlan1_subnet` Sous réseau du raspberry sur le réseau wlan1
- `wlan2_subnet` Sous réseau du raspberry sur le réseau wlan2
- `eth0_dhcp_range_start` Début de plage DHCP sur eth0
- `eth0_dhcp_range_end` Fin de plage DHCP sur eth0
- `wlan1_dhcp_range_start` Début de plage DHCP sur wlan1
- `wlan1_dhcp_range_end` Fin de plage DHCP sur wlan1
- `wlan2_dhcp_range_start` Début de plage DHCP sur wlan2
- `wlan2_dhcp_range_end` Fin de plage DHCP sur wlan2
- `vault_wifis` Tableau de wifi connectable par un appareil, ici utilisé pour le raspberry, pour chaque element, specifier ssid et pass
<!--
- `ip_nas` IP du nas a monter
- `mount_path` Point de montage du nas
- `nas_path` Chemin dans le nas à monter
- `nas_user` Utilisateur cifs du nas
- `vault_nas_pass` Mot de passe cifs du nas
vault_wifi_password: ""
vault_ap_wifi_password: ""
vault_nas_pass: ""
vault_wifis: [
    {'ssid': '', 'pass': ''},
    {'ssid': '', 'pass': ''}
]
-->

### *🔎1

Ces informations sont récupérables à la fin de `Installation du raspberry`

### *🔎2

Pour avoir ces informations il faut utiliser un autre ordinateur  
J'ai personnellement fait une bidouille pour avoir le mac de l'iphone avec le raspberry  
Je ne sais pas si l'adresse mac usb de l'iphone est le même sur tout les ordinateurs  

Alors j'ai fait `network-pre-connect` et `install`  
J'ai connecté l'iphone en partage, l'iphone c'est connecté et j'ai regardé l'adresse mac de celui ci depuis la  

Pour les autres cartes je suis plutot sûr qu'elles ne change pas d'adresse mac entre ordinateurs  
Cela veut dire qu'avec un autre ordinateur, on peut avoir l'adresse mac

### *🔎3

Je suis plutôt sûr que l'adresse mac ne change pas sur des cartes wifi  
Connecter les cartes sur un autre ordinateur et faites un `ifconfig -a`

### Où est l'information d'adresse mac ?

Faites `ifconfig -a` sans avoir connecté la carte  
Puis vous connectez la carte et refaites-le, et essayez de voir quelle carte est nouvelle   
Ici par exemple j'ai isolé wlan1 :

```
wlan1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.3.1  netmask 255.255.255.0  broadcast 192.168.3.255
        inet6 fe80::d221:cc27:4f1b:ab79  prefixlen 64  scopeid 0x20<link>
        ether 00:0f:00:14:63:4f  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 97  bytes 28335 (27.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

On trouve l'adresse mac ici : `ether 00:0f:00:14:63:4f`  
A penser a remplir :
- `ansible/roles/raspberry/templates/etc/hosts.j2` indisponible parce que gitignoré
- Le mot de passe wifi du hotspot `vault_wifi_password` dans .vault
- Le mot de passe wifi des point d'access wifi `vault_ap_wifi_password` dans .vault
- Le reste des informations dans inventory

## Premier tag - network-pre-connect

Sur l'ordinateur, clonez le depot et placez-vous dans le dossier `ansible`  
Je ne vous explique pas comment marche ansible, d'autre tuto existe deja sur internet

```
$ cd ~
$ git clone git@github.com:AtyKlaxas/AnsibleRaspberryBox.git
$ cd AnsibleRaspberryBox/ansible
```

Ensuite lancer le playbook raspberry.yml avec le tag network-pre-connect  
Pensez bien a faire votre fichier inventory et .vault

```
$ ansible-playbook --vault-password-file pass.txt -i inventory raspberry.yml --tags network-pre-connect
```

Maintenant votre raspberry est connecté à internet depuis votre telephone  
Durant toute les operations, laissez votre telephone non connecté en USB comme l'installation le voudrais

## Second tag - install

```
$ ansible-playbook --vault-password-file pass.txt -i inventory raspberry.yml --tags install
```

Maintenant vous avez tous les paquets qu'il nous faudra pour créer la box  
De plus le pilote de la carte wifi est compilé et installé  
⚠️ **Cet operation est longue prévoyez 50 minutes** ⚠️

## Troisième tag - config

```
$ ansible-playbook --vault-password-file pass.txt -i inventory raspberry.yml --tags config
```

Maintenant tout les element à configurer sont configuré  
Il ne manque que de les démarrer

À ce moment vous pouvez connecter l'iphone en partage de connection et les cartes wifi

## Quatrième tag - startup

```
$ ansible-playbook --vault-password-file pass.txt -i inventory raspberry.yml --tags startup
```

Et voilà, raspberry box opérationnel !  
Vous pouvez connecter votre iphone en partage de connection et enlever l'ip fixe de votre ordinateur  
À noter que chez moi j'ai encore des bugs  
ça veut dire que cela peut encore changer d'ici la

## Liens

Ici, des liens qui m'on permit d'apprendre à créer tout ça

1. https://www.framboise314.fr/raspap-creez-votre-hotspot-wifi-avec-un-raspberry-pi-de-facon-express
2. https://www.instructables.com/Share-WiFi-With-Ethernet-Port-on-a-Raspberry-Pi
3. https://doc.ubuntu-fr.org/hostapd
4. https://unix.stackexchange.com/questions/137894/how-do-i-find-out-if-my-wireless-card-supports-5-ghz
5. https://github.com/RinCat/RTL88x2BU-Linux-Driver

Et des tonnes de page stackoverflow

