# Raspberry-SMS
Envoyer et recevoir des SMS depuis le Raspberry Pi

## Matériel : 

Raspberry pi (modèle Zero W ou autre) : 

![](https://www.pi-shop.ch/media/catalog/product/cache/1/image/363x/040ec09b1e35df139433887a97daa66f/z/e/zero_w1.jpg)

Dongle 3G : 

![](https://assistance.orange.fr/medias/woopic/images/var/orange/storage/images/front-prod/equipement/cles-3g-et-dominos/huawei-e352/3629848-93-fre-FR/huawei-e352_full-view-equipment.jpg)

+ Carte micro-SD pour le Raspberry (16 go ?)
+ Chargeur Raspberry pi 
+ Carte Sim (pour des tests, la carte sim proposée chez Free à 2 euros/mois fait très bien l'affaire. De plus, une api est disponible pour envoyer des sms en appelant une url, lien et explications plus bas)

## Première étape : télécharger et installer l'OS 

Télécharger Raspbian : https://www.raspberrypi.org/downloads/raspbian/

Utilitaire : Etcher pour installer Raspbian sur la carte Micro SD : https://www.balena.io/etcher/

## Seconde étape : activer SSH et le Wifi 

Après avoir installé Raspbian, on créé un fichier nommé "ssh" (sans extension), vide, à la racine. 

On créé aussi un fichier wpa_supplicant.conf, avec dedans la configuration du réseau wifi :

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=FR

network={
  ssid="SSID"
  psk="PASSWORD"
  key_mgmt=WPA-PSK
  scan_ssid=1
}
```

## Troisième étape : premier démarrage du Raspberry

carte micro SD installée, Raspberry alimenté, puis : 

Utilitaire angry IP Scanner pour trouver le Raspberry sur le réseau local : http://angryip.org/download/

Mise à jour du système : 

sudo apt update
sudo apt upgrade

## Quatrième étape : Installation de la clé 3G

(Testé avec la clé 3g Huawei E1752)

sudo apt install usb-modeswitch

On branche la clé et on vérifie qu'elle est bien installée : 

dmesg | grep ttyUSB

Création d'un chemin d'accès fixe pour la clé en usb :
On ajoute un second chemin, cette fois-ci fixe, en plus du premier 

lsusb | grep -i huawei

Nous permet de trouver les id recquises : 

Bus 001 Device 007: ID 12d1:1506 Huawei Technologies Co., Ltd. Modem/Networkcard

ici 12d1 et 1506

On fixe le chemin : 

sudo nano /etc/udev/rules.d/98-usb-serial.rules

dedans : 

SUBSYSTEM=="tty", ATTRS{idVendor}=="12d1", ATTRS{idProduct}=="1506", SYMLINK+="ttyUSB-3G"

puis sudo reboot

On vérifie que la clé est visible : 

udevadm info --query=all --name=ttyUSB-3G | grep -i huawei

## Cinquième étape : Installation de Gammu

C'est l'outil qui permet d'envoyer des SMS. 

Génération du fichier de configuration : 

gammu-config -c /home/pi/.gammurc

port : /dev/ttyUSB-3G
Connection : 19200 (115200 fonctionne aussi mais n'apporte rien pour du SMS)
Log File : /ramdisk/gammu.log (si vous avez un ramdisk)
Log format : text
Save pour finir

gammu identify
-> Pour vérifier que la clé 3G est bien reconnue

## Sixième étape : Envoi du premier SMS 

Deux possibilités : 

gammu sendsms TEXT 0612345678 -text "coucou"

ou

echo -e "coucou\nRetour ligne" | gammu -c /home/pi/.gammurc sendsms TEXT 0612345678

Si erreur car besoin du code PIN : 

gammu entersecuritycode PIN 1234

gammu getsmsc 1

## Septième étape : Gammu-SMSD, actions à la réception d'un SMS

sudo apt-get install gammu-smsd

Penser à chmod 777 le repertoire qui gère la réception et l'envoi de sms avec Gammu-SMSD : 

sudo chmod -R 777 /var/spool/gammu/

Fichier de configuration du daemon : /etc/gammu-smsdrc

Doc pour ce fichier : https://wammu.eu/docs/manual/smsd/config.html

dans [SMSD]

on peut rajouter le code pin avec PIN = 
et déclencher un script à la reception d'un SMS avec l'option 
RunOnReceive = /home/pi/monscript.sh

Par défaut, le daemon vérifie toutes les 30 secondes si il a reçu des messages / a des messages à envoyer

On peut modifier ça avec :

CommTimeout = 5
SendTimeout = 5

Envoi de SMS (avec le daemon) : 

gammu-smsd-inject TEXT 06XXXXXXXX -text "Test tutoandco"

ou
echo "Test tutoandco" | gammu-smsd-inject TEXT 06XXXXXX

## Huitième étape : Exemple & Application

Script pour transférer le numéro de l'emetteur du SMS et son message à un script python : 
(à déclarer dans le fichier de configuration dans RunOnReceive)

```
#!/bin/sh
from=$SMS_1_NUMBER
message=$SMS_1_TEXT

python3 /home/pi/SMS.py "$from" "$message"
```

... et dans le script python, on est libre de faire ce qu'on veux :) 

## Neuvième partie : debug / utilitaires 

gestion du service gammu-smsd : start restart stop
sudo service gammu-smsd start

Script pour redémarrer la clé usb (essayer si ça ne fonctionne pas, certains utilisateurs mettent carrément le script dans un crontab) : 

  GNU nano 2.7.4                               File: resetUSB.sh                                         
```
echo "Searching for $1"
devPath=`lsusb | grep $1 | sed -r 's/Bus ([0-9]{3}) Device ([0-9]{3}).*/bus\/usb\/\1\/\2/g;'`
echo "Found $1 @ $devPath"
echo "Searching for sysPath"
for sysPath in /sys/bus/usb/devices/*; do
    echo "$sysPath/uevent"
    devName=`cat "$sysPath/uevent" | grep $devPath`
    #echo devName=$devName
    if [ ! -z $devName ] 
    then
        break
    fi
done
if [ ! -z  $devName ] 
then
    echo "Found $1 @ $sysPath, Resetting"
    echo "echo 0 > $sysPath/authorized"
    echo 0 > $sysPath/authorized
    echo "echo 1 > $sysPath/authorized"
    echo 1 > $sysPath/authorized
else
    echo "Could not find $1"
fi
```

Pour launch : 

/home/pi/resetUSB.sh Huawei

## Sources, Remerciements et Complément d'infos : 

https://projetsdiy.fr/comment-installer-raspbian-raspberry-pi-zero-sans-ecran-clavier/
http://blogmotion.fr/diy/tutoriel-gammu-cle-3g-dongle-16409
https://tutoandco.colas-delmas.fr/software/envoyer-sms-gammu-deamon/

Pour aller plus loin : 

Gestion de la réception directement en python ? https://github.com/faucamp/python-gsmmodem/blob/master/examples/sms_handler_demo.py
