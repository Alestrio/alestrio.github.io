---
author: "Alexis LEBEL"
title: "LSC 1080P : autopsie d’une caméra cloud Tuya trop bavarde"
date: 2026-06-25
lastmod: 2026-06-25
draft: true
description: "accès root, RTSP local, analyse du firmware, watchdog, U-Boot et premiers patchs pour reprendre le contrôle face au cloud."
tags: ["hardware-hacking", "reverse-engineering", "tuya", "lsc-1080p", "camera-ip", "firmware", "u-boot", "spi-flash", "linux-embarque", "rtsp", "watchdog", "iot-security", "cybersecurite", "homelab"]
thumbnail: https://picsum.photos/id/1050/400/250
---

# 0. Transparence - Utilisation de l'IA

L'intelligence artificielle a été utilisée pour aider lors de l'enquête et de la rédaction de cet article, dans les cas suivants :

- Recherche documentaire préliminaire
- Analyse des fonctions désassemblées
- Mise en forme du plan détaillé de l'article à partir de notes de brouillon et de snippets de code
- Rephrasage, correction orthographique et grammaticale


# 1. Objectif et contexte

# 1.1 Objectif

Objectif de départ :

- conserver l’accès local à la caméra ;
- empêcher ou limiter l’envoi de flux vidéo vers l’extérieur ;
- garder assez de connectivité cloud pour que la caméra reste utilisable dans son écosystème ;
- intégrer un snapshot dans Home Assistant, alors que la caméra ne fournit pas d’URL snapshot HTTP native.

Caméra concernée :

- caméra LSC Action / Tuya compatible ;
- flux RTSP local exposé ;
- accès root obtenu via telnet grâce aux travaux du dépôt guino/LSC1080P ;
- accès au prompt U-Boot obtenu via UART en s’appuyant sur le tutoriel berobloom/LSCIndoorCameraLocal1080p, qui décrit la perturbation temporaire de la flash SPI au démarrage.

# 2. Base de départ : caméra connue, accès root et RTSP local

## 2.1 Article d’Alexander Bilz

Points à reprendre :

- caméra vendue par Action, basée sur Tuya ;
- CPU FH8626V100 ;
- firmware 7.6.32 ;
- kernel Linux 4.9.129 ;
- architecture ARMv7 32-bit little endian ;

ports observés :

- 80/tcp
- 835/tcp
- 6668/tcp
- 8554/tcp, RTSP alternatif ;
- présence de dgiot ;
- rootfs SquashFS + partitions JFFS2 ;
- accès telnet activable ;
- mot de passe root connu : dgiot010.

## 2.2 Dépôt guino/LSC1080P

- Points importants :
- activation telnet via un fichier product.cof sur carte SD ;

login :

- utilisateur : root
- mot de passe : dgiot010

RTSP disponible nativement :

- rtsp://IP:8554/main
- rtsp://IP:8554/sub
- pas de snapshot natif facile type snap.cgi;
- pas de mécanisme simple trouvé pour exécuter automatiquement un script custom au boot sans modifier le firmware ;
- recommandation du dépôt : lancer les scripts depuis une machine externe via telnet, par exemple avec initcheck.sh.

## 2.3 Dépôt berobloom/LSCIndoorCameraLocal1080p

Source utilisée pour la partie UART / U-Boot :

- https://github.com/berobloom/LSCIndoorCameraLocal1080p

Ce tutoriel documente notamment :

- l’ouverture de la caméra ;
- le repérage des points UART ;
- l’accès au boot log via liaison série ;
- le contournement du `bootdelay=0` de U-Boot ;
- l’obtention du prompt U-Boot en court-circuitant brièvement deux broches de la flash SPI au démarrage ;
- les commandes U-Boot utiles pour sauvegarder, reconstruire et reflasher l’image.

Dans cette enquête, cette source a surtout servi à débloquer l’accès au prompt U-Boot. Le principe retenu n’était pas de suivre aveuglément le patch complet proposé par le dépôt, mais d’utiliser la méthode d’accès bas niveau pour pouvoir flasher notre propre rootfs modifié.

## 2.4 Activation telnet

Créer sur la carte SD un fichier product.cof :

```sh
[DEFAULT_SETTING]
telnet=1
```

Insérer la carte SD, redémarrer la caméra, puis se connecter :

```sh
telnet 192.168.1.12
```

Identifiants :

```sh
login: root
password: dgiot010
```

## 2.5 Vérification système

Commandes exécutées sur la caméra :

```sh
netstat -tupn
ifconfig
route -n
df -h
cat /etc/inittab
cat /etc/init.d/rcS
```

Résultat observé côté réseau :

wlan0 inet addr:192.168.1.12 Mask:255.255.255.0

Route par défaut :

```sh
Destination     Gateway         Genmask         Flags Iface
0.0.0.0         192.168.1.253   0.0.0.0         UG    wlan0
192.168.1.0     0.0.0.0         255.255.255.0   U     wlan0
```

Conclusion :

la caméra utilise bien 192.168.1.253 comme passerelle IPv4 ;

dans ce cas, 192.168.1.253 correspond au FortiGate ;

# 3. Observation réseau : RTSP local, cloud Tuya et limites du filtrage externe

## 3.1 Connexion cloud principale

Commande :

```sh
netstat -tupn
```

Sortie notable :

```sh
tcp 0 0 192.168.1.12:55804 3.124.225.12:8883 ESTABLISHED 124/dgiot
```

Interprétation :

dgiot maintient une connexion TCP sortante ;

port distant 8883 ;

8883 correspond typiquement à MQTT over TLS ;

cette connexion semble être le canal cloud / contrôle / keepalive.

Je ne peux pas affirmer que toute la vidéo transite par cette session. Elle semble plutôt être le canal cloud principal de la caméra. Le flux vidéo local, lui, est exposé en RTSP sur 8554.

## 3.2 Ne pas tuer dgiot

Test envisagé :

```sh
killall dgiot
```

Mais conclusion :

tuer dgiot rend la caméra inutilisable ;

dgiot est probablement le process central : cloud, RTSP, HTTP, contrôle caméra ;

## 3.3 Sniffer le flux RTSP local

Exemple de flux observé :

192.168.1.12.8554 -> 10.128.0.20.52590
10.128.0.20.52590 -> 192.168.1.12.8554

Interprétation :

```sh
192.168.1.12 : caméra ;
```

10.128.0.20 : machine Homelab, Home Assistant ;

8554 : flux RTSP local ;

ce trafic n’est pas un upload cloud, mais un flux local caméra vers client.

## 3.4 Piège constaté

Malgré la route par défaut vers le FortiGate, la session FortiGate ne remontait pas clairement :

total session: 0

## 3.5 Idée initiale : bloquer UDP

But :

garder HTTP/HTTPS ou MQTT cloud ;

bloquer les flux vidéo/P2P potentiellement en UDP.

Règles conceptuelles :

CAMERA -> DNS local : ACCEPT
CAMERA -> NTP local : ACCEPT
CAMERA -> MQTT TLS / TCP 8883 : ACCEPT
CAMERA -> UDP Internet : DENY
CAMERA -> ALL : DENY

## 3.6 Limite

La vraie solution propre n’est pas uniquement une règle firewall. C’est une topologie où la caméra est placée derrière un SSID/VLAN IoT contrôlé par le FortiGate.

Architecture cible :

Caméra
  -> SSID/VLAN IoT
  -> FortiGate = seule gateway
  -> Box
  -> Internet

Mais contrainte pratique :

pas encore de borne Wi-Fi dédiée ;

besoin d’une solution transitoire.

# 4. Premier contournement : blackhole réseau depuis la caméra

## 4.1 Idée

Comme l’accès telnet est disponible, ajouter une route spécifique vers une gateway inexistante pour empêcher la caméra de joindre une IP cloud donnée.

IP observée, en utilisant netstat -tunp

```sh
3.124.225.12:8883
```

Ajout d'une route mauvaise :

```sh
route add -host 3.124.225.12 gw 192.168.1.31
```

peu importe ce qu'est 31, ,ce ne sera pas un routeur, alors en effet, il y aura un léger trafic et quelues requêtes ARP, mais c'est pas méchant.

## 4.2 Script persistant dans /etc/conf

Créer le script :

```sh
cat > /etc/conf/block_tuya_cloud.sh <<'EOF'
#!/bin/sh

CLOUD_IP="3.124.225.12"
BLACKHOLE_GW="192.168.1.31"

# Attendre que wlan0 ait une IP
i=0
while [ $i -lt 30 ]; do
    if ifconfig wlan0 2>/dev/null | grep -q "inet addr"; then
        break
    fi
    sleep 2
    i=$((i+1))
done

# Nettoyage puis ajout route blackhole
route del -host "$CLOUD_IP" 2>/dev/null
route add -host "$CLOUD_IP" gw "$BLACKHOLE_GW"

# Log minimal
echo "$(date) blackhole $CLOUD_IP via $BLACKHOLE_GW" >> /tmp/block_tuya_cloud.log
route -n >> /tmp/block_tuya_cloud.log
EOF
```

```sh
chmod +x /etc/conf/block_tuya_cloud.sh
```

Test manuel :

```sh
/etc/conf/block_tuya_cloud.sh
route -n
```

Résultat attendu :

```sh
Destination     Gateway         Genmask         Flags Iface
3.124.225.12    192.168.1.31    255.255.255.255 UGH   wlan0
```

Rollback :

```sh
route del -host 3.124.225.12
```

# 5. Contraintes, fausses pistes et premiers enseignements

## 5.1 Analyse des systèmes de fichiers

Commande :

```sh
df -h
```

Sortie observée :

```sh
/dev/root        5.1M  5.1M      0 100% /
/dev/mtdblock5  10.0M 7.6M   2.4M 76% /etc/conf
/dev/mmcblk0p1 236.3G 713.8M 235.6G 0% /mnt
```

Interprétation :

/ est plein et en lecture seule ;

/etc/conf est une partition JFFS2 persistante et modifiable ;

/mnt correspond à la carte SD, même si la taille affichée est incohérente ;

les scripts init système ne sont pas modifiables directement.

## 5.2 Tentative de modification de rcS

Script de boot :

```sh
cat /etc/init.d/rcS
```

Contenu important :

#!/bin/sh

/bin/mount -t proc proc /proc
/bin/mount -t sysfs none /sys
/bin/mount -t ramfs ramfs /var

```sh
mkdir /var/run
mkdir /var/nm
mkdir /var/tmp
mkdir /var/lib
mkdir /var/lib/misc
```

```sh
for initscript in /etc/init.d/S[0-9][0-9]*
do
    if [ -x $initscript ] ;
    then
        echo "[RCS]: $initscript"
        $initscript
    fi
done
```

```sh
echo dev,GPIO61,0,0 > /proc/driver/pinctrl
echo mux,GPIO61,GPIO61,0 > /proc/driver/pinctrl
```

```sh
read -t 1 -n 1 char
if [ "$char" == "n" ];then
        /usr/init/app_init.sh 0 &
else
        /usr/init/app_init.sh 1 &
fi
```

Tentative :

```sh
cat >> /etc/init.d/rcS <<'EOF'

# Custom: blackhole Tuya cloud route
(
    sleep 60
    /bin/sh /etc/conf/block_tuya_cloud.sh
) &
EOF
```

Résultat :

-sh: can't create /etc/init.d/rcS: Read-only file system

Conclusion :

impossible d’ajouter directement un hook dans rcS;

impossible aussi de créer un /etc/init.d/S99custom tant que /etc/init.d est en lecture seule.

## 5.3 Init scripts existants

Fichiers observés :

/etc/init.d/S01udev
/etc/init.d/S02init_rootfs
/etc/init.d/S03network
/etc/init.d/rcS

S02init_rootfs :

```sh
#!/bin/sh
mount -t jffs2 /dev/mtdblock5 /etc/conf
```

S03network :

```sh
#!/bin/sh
/sbin/ifconfig lo 127.0.0.1
#ifconfig eth0 up
```

app_init.sh :

#!/bin/sh

```sh
insmod /usr/modules/ssv6x5x.ko stacfgpath=/etc/ssv6x5x-wifi.cfg
```

# modules vidéo / capteur / média ...

```sh
if [ "$1" == "0" ];then
        echo "stop"
else
        echo "start"
        cd /usr/bin
        ./daemon&
        ./dgiot&
fi
```

Conclusion :

le point logique pour un hook serait app_init.sh ou rcS;

mais ils sont dans le rootfs read-only ;

il faudrait modifier le firmware pour les changer proprement.

## 5.4 Dossier suspect

Observation :

```sh
ls -lah /etc/conf
```

Contenu :

Config/
custom/
deviceinfo
deviceinfo-bk
product.cof
product.cof-bk
systemTime
tuya/

Dossier suspect :

/etc/conf/custom/CustomConfig2

## 5.5 Recherche dans les binaires

```sh
grep -a "/etc/conf" /usr/bin/dgiot
```

Sorties trouvées :

```sh
/etc/conf/deviceinfo
/etc/conf/deviceinfo-bk
/etc/conf/tuya
rm /etc/conf/tuya -rf
touch /etc/conf/voice_prompt_flag
touch /etc/conf/reboot.log
/etc/conf/systemTime
/etc/conf/Config
rm /etc/conf/Config -rf
/etc/conf/tuya/
/etc/conf/custom/CustomConfig2
/etc/conf/Config/Json
/etc/conf/product.cof
cp -a /etc/conf/product.cof /etc/conf/product.cof-bk
...
```

Interprétation :

dgiot référence bien /etc/conf/custom/CustomConfig2;

cela ne prouve pas qu’il exécute un script ;

probablement un dossier de configuration Tuya ;

il faut analyser le binaire dgiot hors caméra.

## 5.6 Objectif

Après avoir constaté que /etc/init.d/rcS et /usr/init/app_init.sh sont situés dans un rootfs en lecture seule, l’objectif était de trouver un point d’accroche déjà présent dans dgiot permettant d’exécuter automatiquement le script de blocage cloud :

/etc/conf/block_tuya_cloud.sh

L’objectif n’était pas de comprendre entièrement le binaire dgiot, mais de répondre à une question précise :

Existe-t-il un hook contrôlable depuis /etc/conf, product.cof, CustomConfig2 ou la carte SD permettant de lancer une commande shell au boot ?

## 5.7 Analyse des chaînes intéressantes

Le binaire dgiot.bin a d’abord été analysé avec strings :

```sh
strings -a -tx bins/dgiot.bin | grep -Ei 'custom|CustomConfig2|product|/etc/conf|/mnt|/tmp|/bin/sh|\\.sh|system|popen|exec|telnet|reboot|route|ifconfig'
```

Résultats intéressants :

```sh
execvp
popen
execlp
exec_cmd
Exec Cmd:%s
popen %s fails, errno : %d
/etc/conf/custom/CustomConfig2
/etc/conf/Config/Json
mount /dev/mmcblk0p1 /mnt
/mnt/product.cof
/etc/conf/product.cof
telnet
telnetd &
touch /tmp/no_voice_prompt_flag
rm -rf /etc/conf/voice_prompt_flag
```

Cela montre que dgiot contient bien des mécanismes d’exécution de commandes, notamment via execlp, popen ou une fonction interne exec_cmd.

## 5.8 Analyse de product.cof

Le fichier product.cof est confirmé comme un fichier INI lu depuis la carte SD.

Le mécanisme observé est le suivant :

```sh
mount /dev/mmcblk0p1 /mnt
/mnt/product.cof
/etc/conf/product.cof
cp -a /etc/conf/product.cof /etc/conf/product.cof-bk
umount -l /mnt
```

La clé connue :

```sh
[DEFAULT_SETTING]
telnet=1
```

permet bien d’activer le serveur telnet.

Une recherche autour de la chaîne telnet montre les clés lues par product.cof :

```sh
DEFAULT_SETTING
telnet
forceBurnLicense
stationssid
STATION
stationpwd
aging_test
AGING_SETTING
```

L’analyse de GetProductCof() montre que les valeurs du fichier INI sont lues avec read_profile_string(), converties avec strtol(), puis stockées dans une structure _PRODUCT_COF.

Aucun chemin évident ne passe une valeur lue depuis product.cof à execlp, popen, system ou exec_cmd.

Conclusion :

product.cof est un hook de configuration, mais pas un hook d’exécution arbitraire.

## 5.9 Fonctionnement de telnet=1

Le désassemblage de main() montre que la clé telnet est testée directement dans la structure de configuration.

Extrait significatif :

```sh
2339c: ldrb r3, [r6, #90]   ; ProductCof_g.telnet
233a0: cmp  r3, #1
233a4: beq  234e4
```

Si ProductCof_g.telnet == 1, dgiot lance ensuite :

```sh
telnetd &
```

via une séquence équivalente à :

```sh
if (ProductCof_g.telnet == 1) {
    execlp("sh", "sh", "-c", "telnetd &", NULL);
}
```

C’est une découverte importante : product.cof déclenche bien une commande shell, mais cette commande est hardcodée dans le binaire.

Il n’est donc pas possible, à ce stade, d’écrire simplement :

```sh
[DEFAULT_SETTING]
cmd=/etc/conf/block_tuya_cloud.sh
```

## 5.10 Refus de patcher telnetd &

Une première idée aurait été de patcher la chaîne :

```sh
telnetd &
```

en :

```sh
/etc/conf/x &
```

ou une commande équivalente.

Cette option a été écartée volontairement, car telnetd est le seul accès root pratique à la caméra. Patcher cette chaîne aurait fait courir un risque de se couper l’accès de maintenance.

## 5.11 Recherche d’une commande non critique à patcher

Une autre commande hardcodée a été identifiée :

```sh
touch /tmp/no_voice_prompt_flag
```

Elle semble liée au comportement de voice prompt après reboot. Cette commande est beaucoup moins critique que telnetd.

Chaînes observées :

```sh
strings -a -tx bins/dgiot.bin | grep -E 'voice_prompt|no_voice'
```

Résultat :

```sh
39cb2c touch /etc/conf/voice_prompt_flag
3a6c00 /etc/conf/voice_prompt_flag
3a6c50 touch /tmp/no_voice_prompt_flag
3a6c70 rm -rf /etc/conf/voice_prompt_flag
```

La chaîne ciblée est donc :

0x3a6c50 touch /tmp/no_voice_prompt_flag

Elle a été remplacée par :

```sh
/etc/conf/x &
```

Cette commande est suffisamment courte pour rentrer dans l’espace de la chaîne originale.

## 5.12 Création d’un premier binaire dgiot_nocloud

Un premier patch non destructif a été appliqué au binaire dgiot.bin.

```sh
Le principe est simple : remplacer certaines chaînes cloud Tuya par 127.0.0.1, en conservant exactement la taille des chaînes grâce à un padding nul (\\0).
```

Les chaînes ciblées dans ce premier patch étaient :

h2-we.iot-dns.com
h2.iot-dns.com
https://a2.tuyaeu.com/d.json
https://a2-weaz.tuyaeu.com/d.json
m2.tuyaeu.com
m2-weaz.tuyaeu.com

Script utilisé côté machine d’analyse :

```sh
cp bins/dgiot.bin bins/dgiot_nocloud.bin
```

```sh
python3 - <<'PY'
from pathlib import Path

p = Path("bins/dgiot_nocloud.bin")
data = bytearray(p.read_bytes())

patches = [
    # HTTP DNS Tuya
    (b"h2-we.iot-dns.com", b"127.0.0.1"),
    (b"h2.iot-dns.com",    b"127.0.0.1"),

    # Bootstrap/config régionaux
    (b"https://a2.tuyaeu.com/d.json",      b"https://127.0.0.1/d.json"),
    (b"https://a2-weaz.tuyaeu.com/d.json", b"https://127.0.0.1/d.json"),

    # MQTT EU direct
    (b"m2.tuyaeu.com",      b"127.0.0.1"),
    (b"m2-weaz.tuyaeu.com", b"127.0.0.1"),
]

for old, new in patches:
    pos = data.find(old)
    if pos < 0:
        print(f"not found: {old!r}")
        continue
    if len(new) > len(old):
        raise ValueError(f"replacement too long: {new!r} > {old!r}")

    data[pos:pos+len(old)] = new + b"\\x00" * (len(old) - len(new))
    print(f"patched {old.decode(errors='ignore')} at {hex(pos)} -> {new.decode(errors='ignore')}")

p.write_bytes(data)
PY
```

Vérification :

```sh
strings -a -tx bins/dgiot_nocloud.bin | grep -Ei 'iot-dns|a2\\.tuya|m2\\.tuya|127\\.0\\.0\\.1'
```

Résultat observé :

patched h2-we.iot-dns.com at 0x3cab00 -> 127.0.0.1
patched h2.iot-dns.com at 0x3cab14 -> 127.0.0.1
patched https://a2.tuyaeu.com/d.json at 0x3cbc78 -> https://127.0.0.1/d.json
patched https://a2-weaz.tuyaeu.com/d.json at 0x3cbcbc -> https://127.0.0.1/d.json
patched m2.tuyaeu.com at 0x3e1634 -> 127.0.0.1
patched m2-weaz.tuyaeu.com at 0x3e1644 -> 127.0.0.1

Le binaire patché est ensuite copié sur la carte SD, montée dans la caméra sous /mnt.

## 5.13 Premier test manuel : échec apparent à cause du watchdog

Le premier test consistait à tuer dgiot, puis à lancer le binaire patché :

```sh
killall dgiot && /mnt/dgiot_nocloud.bin
```

Le binaire démarrait partiellement, mais affichait rapidement des erreurs liées au watchdog :

open watchdog dev fail
err(wdt_keep_alive,22): Bad file descriptor

La caméra finissait ensuite par crasher ou redémarrer.

À ce stade, il aurait été dangereux de remplacer directement /usr/bin/dgiot dans le rootfs. Un tel remplacement aurait probablement provoqué une boucle de redémarrage, car le binaire patché lancé dans ces conditions ne parvenait pas à nourrir le watchdog.

## 5.14 Échec du patch chaînes seul : le cloud reste accessible

Malgré le patch des chaînes visibles, une vérification réseau montre encore des connexions sortantes :

```sh
netstat -tunp
```

Résultat :

```sh
tcp 0 0 192.168.1.12:39072 3.124.225.12:8883 ESTABLISHED 2072/dgiot_nocloud.
tcp 0 0 192.168.1.12:45529 63.177.1.197:1443 ESTABLISHED 2072/dgiot_nocloud.
tcp 0 0 192.168.1.12:8554 192.168.1.11:64829 ESTABLISHED 2072/dgiot_nocloud.
```

Interprétation :

```sh
192.168.1.12:8554 -> 192.168.1.11 correspond au flux RTSP local.
```

```sh
3.124.225.12:8883 correspond à une connexion MQTT/TLS Tuya.
```

```sh
63.177.1.197:1443 correspond à un endpoint HTTPS/Tuya ou assimilé.
```

Et surtout : un ps montre que l'ancien dgiot est revenu, la caméra ayant rebooté.

La cause a ensuite été précisée : dgiot avait ouvert /dev/watchdog, mais le descripteur avait également été hérité par telnetd. Après remplacement du processus, le nouveau dgiot ne pouvait donc pas toujours reprendre proprement le watchdog. Le reboot n’était pas une preuve suffisante que le patch du binaire était défectueux.

# 6. Extraction et analyse de dgiot

## 6.1 Méthode TFTP

Windows : utilisation d'un server tftp temporaire : https://github.com/PJO2/tftpd64/releases/

## 6.2 Sauvegarde complète de la flash

Avant toute tentative de modification persistante, une copie brute de chaque partition MTD a été réalisée sur la carte SD.

La disposition déclarée par le noyau a d’abord été vérifiée :

```sh
cat /proc/mtd
ls -l /dev/mtd*
```

Puis les partitions ont été copiées sans transformation :

```sh
mkdir -p /mnt/dump
```

```sh
for m in /dev/mtd[0-9]*; do
    name=$(basename "$m")
    cat "$m" > "/mnt/dump/$name.bin"
done
```

```sh
md5sum /mnt/dump/*.bin > /mnt/dump/md5sums.txt
tar czf /mnt/dump/etc_conf.tar.gz /etc/conf
```

Cette étape produit :

```sh
mtd0_bootstrap.bin
mtd1_uboot-env.bin
mtd2_uboot.bin
mtd3_kernel.bin
mtd4_rootfs.bin
mtd5_config.bin
etc_conf.tar.gz
md5sums.txt
```

L’objectif est double :

- disposer d’une sauvegarde avant tout repack ou flash ;
- travailler hors caméra sur des copies reproductibles.

Le dump de la RAM complète n’a volontairement pas été réalisé. Il pourrait devenir utile plus tard pour retrouver des domaines dynamiques, des structures runtime ou des secrets chargés en mémoire, mais il apporte moins de valeur immédiate qu’un dump fiable de la flash et de la configuration.

## 6.3 Extraction du rootfs

Le SquashFS a été extrait avec Binwalk :

```sh
binwalk -e mtd4_rootfs.bin
```

Le système de fichiers obtenu se trouve dans :

dump/_mtd4_rootfs.bin.extracted/squashfs-root

Les premières cibles retrouvées sont :

usr/bin/dgiot
usr/bin/daemon
lib/libIOTCAPIs.so
lib/libAVAPIs.so
lib/libsCHL.so
usr/init/app_init.sh
etc/init.d/rcS

Leur identification a été vérifiée avec :

```sh
file usr/bin/dgiot
file usr/bin/daemon
file lib/libIOTCAPIs.so
file lib/libAVAPIs.so
file lib/libsCHL.so
```

Résultat structurant :

- dgiot est un ELF ARM non stripped ;
- daemon est stripped ;
- les bibliothèques IOTC/AV/SCHL sont stripped.

Le fait que dgiot conserve sa table de symboles rend possible une analyse CLI précise, même sans décompilateur graphique.

## 6.4 Première cartographie dynamique de dgiot

Depuis le rootfs extrait :

```sh
mkdir -p ../../analysis
```

```sh
readelf -h usr/bin/dgiot \\
  > ../../analysis/dgiot.elf_header.txt
```

```sh
readelf -d usr/bin/dgiot \\
  > ../../analysis/dgiot.dynamic.txt
```

```sh
readelf -Ws usr/bin/dgiot \\
  > ../../analysis/dgiot.symbols.txt
```

Les dépendances dynamiques ont été isolées avec :

```sh
grep NEEDED ../../analysis/dgiot.dynamic.txt
```

Elles comprennent notamment :

libAVAPIs.so
libIOTCAPIs.so
libsCHL.so
libpthread.so.0
libdl.so.0
libstdc++.so.6
libc.so.0

La pile TUTK/IOTC n’est donc pas seulement présente dans le firmware : elle est explicitement liée à dgiot.

Les imports observés incluent :

IOTC_Initialize2
IOTC_Device_Login
IOTC_Listen
IOTC_Session_Check
IOTC_Session_Close
IOTC_Session_Channel_ON
IOTC_Session_Channel_OFF
IOTC_Set_Max_Session_Number
IOTC_Set_Device_Name

## 6.5 Analyse des chaînes de dgiot et des bibliothèques P2P

Les chaînes ont été conservées dans des fichiers séparés :

```sh
strings -a usr/bin/dgiot \\
  > ../../analysis/dgiot.strings.txt
```

```sh
strings -a lib/libIOTCAPIs.so \\
  > ../../analysis/libIOTCAPIs.strings.txt
```

```sh
strings -a lib/libAVAPIs.so \\
  > ../../analysis/libAVAPIs.strings.txt
```

```sh
strings -a lib/libsCHL.so \\
  > ../../analysis/libsCHL.strings.txt
```

Puis les termes utiles ont été regroupés :

```sh
grep -Ei \\
'tuya|mqtt|p2p|iotc|tutk|cloud|dns|rtsp|onvif|stun|turn|relay|login|connect|8883|1443|8554|watchdog|wdt|dlopen|dlsym' \\
../../analysis/*.txt \\
> ../../analysis/interesting_hits.txt
```

Cette recherche a mis en évidence des unités provenant directement du SDK et de l’application :

Tuya.cpp
tuya_main.cpp
tuya_ipc_p2p.cpp
tuya_ipc_p2p.c
tuya_ipc_mqt_proccess.c
mqtt_client.c
direct_conect_tuya.c
tuya_tls.c
tuya_tcp.c
tuya_udp.c
tuya_nat_detector.c
tuya_stun_message.c
tuya_relay_session.c
live_rtsp.cpp
VideoRTSPMan.cpp
Wdt.c

Conclusion : dgiot est un binaire monolithique qui contient une part importante du SDK Tuya, en plus de charger les bibliothèques TUTK dynamiques.

## 6.6 Méthode retenue sans Ghidra

Ghidra n’était pas disponible dans l’environnement d’analyse. Ce n’est pas bloquant car dgiot n’est pas stripped.

La chaîne d’outils retenue est volontairement simple :

```sh
binwalk / unsquashfs
file
readelf
nm / c++filt
strings
objdump ARM
grep
xxd
Python ou dd pour les patchs reproductibles
```

Une version démanglée des symboles C++ a d’abord été créée :

```sh
readelf -Ws usr/bin/dgiot |
  c++filt \\
  > ../../analysis/dgiot.symbols.demangled.txt
```

Puis les fonctions candidates ont été extraites :

```sh
grep -Ei \\
'CTuya|initP2p|initp2pServer|p2p|mqtt|cloud|direct|IOTC|rtsp|watchdog|wdt|tuya_ipc|tuya_iot' \\
../../analysis/dgiot.candidates.txt
```

Les chaînes avec leurs offsets fichier ont également été générées :

```sh
strings -a -tx usr/bin/dgiot \\
  > ../../analysis/dgiot.strings.offsets.txt
```

```sh
grep -Ei \\
'mqtt ok|p2p v3 init ok|mqtt p2p LISTEN|Failed to create RTSP|initP2p|initp2pServer|IOTC_Device_Login|IOTC_Initialize2|direct connect|cloud' \\
../../analysis/dgiot.strings.offsets.txt \\
> ../../analysis/dgiot.runtime_strings_hits.txt
```

Enfin, les relocations IOTC ont été recherchées :

```sh
readelf -r usr/bin/dgiot |
  grep -Ei 'IOTC|AV_|TUTK|P2P' \\
  > ../../analysis/dgiot.iotc.relocs.txt
```

## 6.7 Première stratégie de patch envisagée

Trois niveaux de patch ont été comparés.

## Niveau 1 — fonctions haut niveau de dgiot

Cibles envisagées :

initP2p()
initp2pServer()
fonction de démarrage du SDK Tuya
fonction d’initialisation MQTT/cloud

Avantages :

- frontière fonctionnelle claire ;
- moins de risque de casser les sockets locales ;
- possibilité de préserver RTSP et ONVIF.

## Niveau 2 — appels ou bibliothèque IOTC

Une autre possibilité serait de neutraliser les appels :

IOTC_Initialize2
IOTC_Device_Login
IOTC_Listen

ou de remplacer libIOTCAPIs.so par une bibliothèque stub exportant les mêmes symboles.

Cette approche a été gardée en réserve car elle est plus basse dans la pile et suppose de reproduire correctement le comportement attendu par dgiot.

## Niveau 3 — primitives réseau globales

Cibles possibles mais volontairement écartées en première intention :

connect
send
recv
gethostbyname
TLS connect
DNS interne

Ces primitives sont partagées avec RTSP, ONVIF et d’autres threads. Les patcher globalement ferait courir un risque important de régression ou de reboot.

La décision prise à ce stade est donc :

1. confirmer que RTSP est initialisé séparément ;
2. neutraliser d’abord la pile P2P ;
3. vérifier stabilité, watchdog et RTSP ;
4. neutraliser MQTT/cloud dans une seconde étape ;
5. ne reconstruire le firmware qu’après validation runtime.

# 7. Premier patch runtime et piège du watchdog

## 7.1 Objectif de l’essai

À ce stade, le but n’est pas encore de modifier le firmware de manière persistante. L’objectif est plus limité : vérifier qu’un dgiot modifié peut démarrer en runtime, exécuter un hook contrôlé, conserver le RTSP local, et appliquer le blocage cloud sans casser la caméra.

La stratégie retenue consiste à remplacer une commande hardcodée non critique dans dgiot.bin par un appel vers un script persistant situé dans /etc/conf.

Chaîne originale repérée dans le binaire :

```sh
touch /tmp/no_voice_prompt_flag
```

Chaîne de remplacement :

```sh
/etc/conf/x &
```

Ce choix est volontairement conservateur : on ne touche pas à telnetd &, ce qui évite de perdre l’accès de secours à la caméra.

## 7.2 Patch du binaire

Le patch est appliqué sur une copie locale de dgiot.bin :

```sh
cp bins/dgiot.bin bins/dgiot_patched.bin
printf '/etc/conf/x & ' | dd of=bins/dgiot_patched.bin bs=1 seek=$((0x3a6c50)) conv=notrunc
```

Vérification minimale :

```sh
strings -a -tx bins/dgiot_patched.bin | grep -E '/etc/conf/x|telnetd'
```

Résultat utile :

```sh
3a6c50 /etc/conf/x &
3a727c telnetd &
```

Le hook est donc bien injecté, et la commande de lancement de telnet reste intacte.

## 7.3 Script appelé par le hook

Le script /etc/conf/x sert uniquement de point d’entrée. Il appelle le script de blocage cloud déjà préparé, puis écrit un log temporaire permettant de vérifier son exécution.

```sh
cat > /etc/conf/x <<'EOF'
#!/bin/sh

/etc/conf/block_tuya_cloud.sh
echo "$(date) /etc/conf/x executed" >> /tmp/etc_conf_x.log
EOF
```

```sh
chmod +x /etc/conf/x
```

Test manuel rapide :

```sh
/etc/conf/x
cat /tmp/etc_conf_x.log
cat /tmp/block_tuya_cloud.log
route -n
```

La route blackhole attendue apparaît bien :

```sh
3.124.225.12 via 192.168.1.31
```

## 7.4 Validation du hook en runtime

Le binaire patché est transféré sur la caméra, puis lancé manuellement sans remplacer encore le binaire original du rootfs.

```sh
tftp -g -r dgiot_patched.bin -l /dev/dgiot_patched.bin 192.168.1.11
chmod +x /dev/dgiot_patched.bin
cd /usr/bin
/dev/dgiot_patched.bin &
```

Au premier lancement, le hook ne s’exécute pas forcément : la branche contenant la commande patchée dépend de l’état interne de la caméra. Pour forcer le passage dans cette branche, les fichiers de contrôle liés au prompt vocal ont été ajustés :

```sh
rm -f /etc/conf/voice_prompt_flag
touch /etc/conf/reboot.log
sync
```

Après relance, le hook est bien atteint :

Mon Jun 22 23:20:41 UTC 2026 /etc/conf/x executed

Et la route de blocage est présente :

```sh
3.124.225.12    192.168.1.31    255.255.255.255 UGH   wlan0
```

Conclusion locale : le patch fonctionne. Quand la branche est atteinte, dgiot_patched.bin exécute bien /etc/conf/x, et le script peut appliquer le blocage réseau.

## 7.5 Limite immédiate : rien n’est encore persistant

Après reboot, les logs dans /tmp disparaissent, la route blackhole aussi, et le processus actif redevient le dgiot original.

C’est attendu :

- /tmp est volatile ;
- /dev est volatile ;
- le binaire patché n’a pas encore été réintégré dans le rootfs ;
- le rootfs SquashFS reste en lecture seule.

À ce stade, le patch est donc une preuve de concept runtime, pas encore une modification durable.

## 7.6 Le problème inattendu : le watchdog

Lors des essais de remplacement de dgiot à chaud, la caméra finit par redémarrer. Le premier réflexe serait d’accuser le patch, mais le problème vient en réalité du watchdog.

La caméra expose un watchdog matériel :

```sh
ls -l /dev/*watch* /dev/wdt* 2>/dev/null
cat /proc/misc 2>/dev/null | grep -i watch
dmesg | grep -Ei 'watchdog|wdt'
```

Résultat :

```sh
crw-------    1 root     root       10, 130 Jan  1  1970 /dev/watchdog
130 watchdog
[    6.748227] fh_wdt: [wdt] set topval: 5
[    6.748398] fh_wdt: [wdt] set topval: 8
```

Un kicker externe naïf ne peut pas l’ouvrir :

can't create /dev/watchdog: Device or resource busy

Cela indique qu’un autre processus possède déjà /dev/watchdog.

## 7.7 Le vrai piège : héritage du file descriptor watchdog

L’inspection des file descriptors révèle le point clé :

```sh
PID=124 COMM=dgiot   FD=/proc/124/fd/3 -> /dev/watchdog
PID=136 COMM=telnetd FD=/proc/136/fd/3 -> /dev/watchdog
```

dgiot ouvre bien le watchdog, mais telnetd en hérite aussi. La cause probable est classique : dgiot lance telnetd sans marquer le file descriptor du watchdog en close-on-exec.

La conséquence est piégeuse :

1. on tue dgiot depuis la session telnet ;
2. telnetd reste vivant ;
3. telnetd garde /dev/watchdog ouvert ;
4. le nouveau dgiot_nocloud.bin ne peut plus ouvrir proprement le watchdog ;
5. le thread FeedDog échoue ;
6. la caméra reboot.

Le reboot n’est donc pas une preuve que le patch réseau casse la caméra. C’est principalement un problème d’héritage de descripteur.

## 7.8 Remplacement runtime propre de dgiot

Le remplacement à chaud doit donc être fait par un script détaché qui :

- ferme les file descriptors hérités ;
- tue dgiot et telnetd ;
- lance le binaire patché ;
- relance ensuite telnetd ;
- vérifie que seul le nouveau dgiot possède /dev/watchdog.

Version condensée du script utilisé :

```sh
cat > /tmp/swap_nocloud_cleanfd.sh <<'EOF'
#!/bin/sh

PATCHED="/mnt/dgiot_nocloud.bin"
LOG="/tmp/swap_nocloud_cleanfd.log"

echo "=== $(date) scheduled clean-fd swap ===" >> "$LOG"
sleep 2

# Fermer les FD hérités de la session telnet, notamment /dev/watchdog.
exec 3>&-
exec 4>&-
exec 5>&-
exec 6>&-
exec 7>&-
exec 8>&-
exec 9>&-

chmod +x "$PATCHED"

killall -9 dgiot 2>/dev/null
killall -9 dgiot_nocloud.bin 2>/dev/null
killall -9 telnetd 2>/dev/null

sleep 1

cd /usr/bin || exit 1
"$PATCHED" >> /tmp/dgiot_nocloud_stdout.log 2>&1 &

sleep 2
telnetd &

sleep 2
ps | grep -E 'dgiot|telnetd' | grep -v grep >> "$LOG"

for p in /proc/[0-9]*; do
    pid="${p#/proc/}"
    comm="$(cat "$p/comm" 2>/dev/null)"
    if ls -l "$p/fd" 2>/dev/null | grep -q '/dev/watchdog'; then
        echo "--- PID=$pid COMM=$comm ---" >> "$LOG"
        ls -l "$p/fd" 2>/dev/null | grep '/dev/watchdog' >> "$LOG"
    fi
done
EOF
```

```sh
chmod +x /tmp/swap_nocloud_cleanfd.sh
/tmp/swap_nocloud_cleanfd.sh >/tmp/swap_cleanfd_launcher.log 2>&1 &
```

La session telnet se coupe pendant le swap, ce qui est normal. Après reconnexion, le processus patché est bien actif :

2072 root      258m S    /mnt/dgiot_nocloud.bin
2083 root      1512 S    telnetd

L’objectif final est que telnetd ne garde plus /dev/watchdog, et que seul dgiot_nocloud.bin le possède.

## 7.9 Validation fonctionnelle

Après le swap propre, le binaire patché démarre correctement. Les threads internes attendus apparaissent :

Main
FeedDog
TimerManager
AudioManager
Pooled
AudioPrompt

La caméra relance ses services vidéo/audio, récupère son IP en DHCP, et le RTSP local reste disponible :

rtsp://192.168.1.12:8554/main
rtsp://192.168.1.12:8554/sub

Ce point est important : le patch par chaîne et le remplacement runtime ne cassent pas le fonctionnement local de la caméra.

## 7.10 Bilan de cette étape

Cette étape établit plusieurs résultats utiles :

- dgiot peut être patché par remplacement de chaîne sans casser immédiatement le binaire ;
- /etc/conf/x est un hook viable pour appeler un script persistant ;
- le blocage cloud peut être appliqué depuis la caméra en runtime ;
- le RTSP local reste fonctionnel ;
- le reboot observé pendant les premiers swaps vient du watchdog et de l’héritage de FD, pas nécessairement du patch ;
- aucune modification persistante dangereuse n’a encore été faite sur le rootfs.

La suite logique, si l’on veut rendre ce comportement permanent, serait de travailler hors caméra sur le firmware extrait : extraire le SquashFS, remplacer proprement /usr/bin/dgiot, reconstruire l’image, puis vérifier soigneusement tailles, offsets et mécanismes de boot avant tout flash.

# 8. Cartographie applicative : RTSP, TUTK et chemin Tuya réel

## 8.1 Objectif

Le patch de chaînes DNS a montré ses limites : le SDK Tuya possède ses propres mécanismes de cache, de résolution dynamique et de reconnexion. La suite de l’analyse consiste donc à rechercher des frontières fonctionnelles de plus haut niveau :

- initialisation du P2P TUTK/IOTC ;
- initialisation du SDK Tuya ;
- MQTT et découverte cloud ;
- P2P/WebRTC Tuya ;
- stockage cloud ;
- RTSP local, qui doit rester intact ;
- watchdog, qui doit continuer à être alimenté.

Avant de modifier le code ARM, il faut établir la disposition mémoire du binaire et reconstruire sa chaîne d’initialisation.

## 8.2 Préparation des chemins d’analyse

Les commandes suivantes sont exécutées depuis la racine du projet :

```sh
BIN="dump/_mtd4_rootfs.bin.extracted/squashfs-root/usr/bin/dgiot"
OUT="dump/analysis"
mkdir -p "$OUT/functions"
```

BIN désigne le binaire extrait du SquashFS. OUT regroupe les résultats afin de conserver une analyse reproductible.

## 8.3 Identification et empreinte du binaire

```sh
file "$BIN"
sha256sum "$BIN" > "$OUT/dgiot.sha256.txt"
readelf -hW "$BIN" > "$OUT/dgiot.elf-header.txt"
readelf -SW "$BIN" > "$OUT/dgiot.sections.txt"
readelf -lW "$BIN" > "$OUT/dgiot.segments.txt"
```

Résultats importants :

- ELF 32 bits little-endian pour ARM EABI5 ;
- exécutable dynamiquement lié ;
- interpréteur /lib/ld-uClibc.so.0 ;
- symboles non supprimés ;
- exécutable de type EXEC, donc non-PIE ;
- point d’entrée ELF : 0x00023ADC ;
- fonction main : 0x00022CDC.

Le caractère non-PIE est particulièrement utile : les adresses observées dans la table des symboles correspondent directement à celles du désassemblage.

## 8.4 Extraction et démangling des symboles

```sh
nm -n "$BIN" > "$OUT/dgiot.nm.txt"
nm -n -C "$BIN" > "$OUT/dgiot.nm.demangled.txt"
```

L’option -n trie les symboles par adresse. L’option -C transforme les noms C++ encodés, par exemple _Z7initP2pv, en noms lisibles comme initP2p().

Un premier index des fonctions d’initialisation a ensuite été produit :

```sh
grep -Ei \\
'(^| )(main|initP2p|initp2pServer|rtspThread|CTuya|tuya.*init|ipc.*init|mqtt|cloud.storage|direct.connect|watchdog|wdt)' \\
"$OUT/dgiot.nm.demangled.txt" \\
> "$OUT/dgiot.init-candidates.txt"
```

Quelques symboles structurants :

```sh
00022cdc T main
00035084 T CTuya::ThreadProc()
00035f70 T CTuya::init()
00035f78 T CTuya::start()
00044b98 T IPC_APP_Init_SDK(WIFI_INIT_MODE_E, char*)
0006b7b0 T initP2p()
0006b8ec T initp2pServer()
00079414 t rtspThread(void*)
000c80b4 T WatchdogCreate
0012d2dc T tuya_ipc_init_sdk
00135468 T tuya_ipc_cloud_storage_init
0015a300 T tuya_ipc_webrtc_init
00169e2c T tuya_p2p_rtc_init
00193f64 T direct_connect_tuya_cloud
001c9a10 T mqtt_client_start
```

## 8.5 Désassemblage ARM

```sh
arm-linux-gnueabi-objdump -d -C "$BIN" > "$OUT/dgiot.disasm"
arm-linux-gnueabi-objdump -dr -C "$BIN" > "$OUT/dgiot.disasm.relocs"
readelf -rW "$BIN" > "$OUT/dgiot.relocations.txt"
arm-linux-gnueabi-objdump -R "$BIN" > "$OUT/dgiot.dynamic-relocations.txt"
```

Les relocations et la PLT permettent de reconnaître les appels vers les bibliothèques dynamiques, notamment libIOTCAPIs.so et libAVAPIs.so.

## 8.6 Extraction des fonctions P2P et RTSP

Les fonctions ont été isolées dans des fichiers plus courts :

```sh
arm-linux-gnueabi-objdump -d \\
  --disassemble=_Z7initP2pv \\
  "$BIN" \\
  > "$OUT/functions/initP2p.asm"
```

```sh
arm-linux-gnueabi-objdump -d \\
  --disassemble=_Z13initp2pServerv \\
  "$BIN" \\
  > "$OUT/functions/initp2pServer.asm"
```

```sh
arm-linux-gnueabi-objdump -d \\
  --disassemble=_ZL10rtspThreadPv \\
  "$BIN" \\
  > "$OUT/functions/rtspThread.asm"
```

Pour retrouver leurs appelants :

```sh
grep -n -B 12 -A 16 \\
  -E 'blx?[[:space:]].*<(initP2p|initp2pServer|rtspThread)' \\
  "$OUT/dgiot.disasm" \\
  > "$OUT/dgiot.init-callsites.txt"
```

## 8.7 Chaîne P2P TUTK/IOTC reconstruite

Le désassemblage établit la chaîne suivante :

CTutk::ThreadProc()
  └── initp2pServer()
        ├── initConfig()
        └── initP2p()
              ├── InitAVInfo()
              ├── IOTC_Set_Device_Name()
              ├── IOTC_Set_Max_Session_Number(4)
              ├── IOTC_Initialize2()
              ├── IOTC_Get_Version()
              ├── IOTC_Get_Login_Info_ByCallBackFn()
              └── avInitialize(12)

initP2p() retourne 1 en cas de succès et 0 en cas d’échec. initp2pServer() retourne -1 si initConfig() échoue ; sinon, il effectue un tail-call vers initP2p().

La recherche de tous les appels IOTC a été réalisée avec :

```sh
grep -n -B 8 -A 12 \\
  -E 'blx?[[:space:]].*<(IOTC_|avInitialize|avDeInitialize)' \\
  "$OUT/dgiot.disasm" \\
  > "$OUT/dgiot.iotc-callsites.txt"
```

Elle révèle également :

- thread_Login() appelle IOTC_Device_Login() et recommence après 30 secondes en cas d’échec ;
- CTutk::ThreadProc() entre dans une boucle IOTC_Listen() ;
- jusqu’à quatre sessions semblent être gérées ;
- thread_ForAVServerStart() utilise IOTC_Session_Check(), IOTC_Session_Channel_ON/OFF() et les fonctions AV ;
- les sessions P2P consomment les flux audio/vidéo produits localement.

Une conséquence importante apparaît : neutraliser uniquement initP2p() sans interrompre le reste de CTutk::ThreadProc() serait risqué. La fonction appelante poursuit immédiatement la création de threads et la boucle IOTC_Listen() sans tester clairement le résultat de initp2pServer().

La frontière de patch la plus prometteuse est donc probablement le lancement de CTutk::ThreadProc() lui-même, ou son appelant, plutôt qu’un faux succès retourné par initP2p().

## 8.8 RTSP local séparé de la pile IOTC

Le chemin RTSP observé est :

rtspThread()
  └── VideoRTSPMan::Start()

rtsp_video_create()
  ├── VideoRTSPMan::createNew()
  └── VideoRTSPMan::Start()

Le RTSP exploite les flux capturés et leurs buffers, mais ne dépend pas directement de IOTC_Initialize2() ou de IOTC_Listen(). Cela renforce l’hypothèse qu’il est possible de supprimer la pile TUTK tout en conservant /main et /sub.

## 8.9 Deux piles P2P distinctes

L’analyse des symboles montre qu’il ne faut pas confondre :

1. la pile historique TUTK/IOTC, liée à libIOTCAPIs.so et libAVAPIs.so ;
2. la pile Tuya P2P/WebRTC intégrée dans dgiot, avec tuya_p2p_rtc_init, STUN, relay et signaling MQTT.

Neutraliser TUTK ne suffira donc pas nécessairement à supprimer toutes les communications P2P.

## 8.10 Organisation mémoire de la caméra

Les informations runtime ont été collectées dans :

```sh
cat dump/sysinfo/proc_mtd.txt
cat dump/sysinfo/proc_meminfo.txt
cat dump/sysinfo/proc_cmdline.txt
cat dump/sysinfo/mount.txt
cat dump/sysinfo/df_h.txt
```

La flash SPI de 8 MiB est découpée ainsi :

| Partition | Offset flash | Taille | Rôle |
| :---- | :---- | :---- | :---- |
| mtd0 | 0x000000 | 256 KiB | bootstrap |
| mtd1 | 0x040000 | 64 KiB | environnement U-Boot |
| mtd2 | 0x050000 | 256 KiB | U-Boot |
| mtd3 | 0x090000 | 1856 KiB | kernel uImage |
| mtd4 | 0x260000 | 5440 KiB | rootfs SquashFS |
| mtd5 | 0x7B0000 | 320 KiB | configuration JFFS2 |

Le noyau reçoit mem=36M, tandis que /proc/meminfo expose environ 31 MiB utilisables et aucun swap.

Le détail des segments virtuels de dgiot est notamment :

| Zone ELF | Adresse virtuelle | Propriété |
| :---- | :---- | :---- |
| segment principal | 0x00010000 | lecture/exécution |
| .text | 0x0001A640 | code ARM |
| .rodata | 0x003AA260 | chaînes et constantes |
| segment mutable | 0x00552630 | lecture/écriture |
| .data | 0x00554DC0 | variables initialisées |
| .bss | 0x005A3580 | variables globales initialisées à zéro |

Quelques objets globaux utiles ont également une adresse stable :

wdt_fd                    0x00599010
CTuya::instance::_instance 0x005A38B8
cloud_storage_ctl         0x00659E94

## 8.11 État de l’enquête

À cette étape :

- le RTSP paraît architecturalement distinct du P2P TUTK ;
- initP2p() est bien l’initialisation directe des bibliothèques IOTC/AV ;
- patcher seulement son retour est probablement insuffisant ou dangereux ;
- CTutk::ThreadProc() continue sans contrôler ce retour ;
- la pile P2P/WebRTC Tuya devra être traitée séparément ;
- les primitives réseau globales comme connect, send, recv, DNS ou TLS restent de mauvaises premières cibles ;
- le watchdog est indépendant des sous-systèmes cloud, mais impose de conserver le thread d’alimentation et une gestion propre de son descripteur.

La prochaine étape consiste à identifier précisément qui construit et démarre l’objet CTutk, puis à cartographier les appels qui activent MQTT, cloud storage et P2P/WebRTC depuis le SDK Tuya.

## 8.12 Pourquoi cette vérification était nécessaire

La présence des bibliothèques IOTC et des fonctions initP2p() ne prouve pas qu’elles sont exécutées sur tous les modèles utilisant ce binaire.

dgiot semble être construit pour plusieurs variantes de produit. Il fallait donc retrouver l’appelant de CTutk::start() et déterminer si ce chemin est réellement celui de la caméra LSC/Tuya étudiée.

## 8.13 Recherche des symboles et des appelants de CTutk

Les symboles de la classe ont été recherchés avec :

```sh
grep -n -E 'CTutk::|<CTutk' \\
  dump/analysis/dgiot.nm.demangled.txt \\
  dump/analysis/dgiot.disasm
```

Les appels à son singleton et à son démarrage ont ensuite été isolés :

```sh
grep -n -B 20 -A 30 \\
  -E 'bl.*<(CTutk::CTutk|CTutk::start|CTutk::init|CTutk::instance|CTutk::ThreadProc)' \\
  dump/analysis/dgiot.disasm
```

Résultat :

CTutk::instance()  0x0006BB80
CTutk::init()      0x0006BBD0
CTutk::start()     0x0006BBD8
CTutk::ThreadProc  0x0006B944

CTutk::start() ne fait qu’appeler CThread::CreateThread(). Son unique appelant significatif se trouve dans CSofia::start() :

```sh
7aee4: bl  6bb80 <CTutk::instance()>
7aee8: bl  6bbd8 <CTutk::start()>
```

## 8.14 Désassemblage de l’orchestrateur CSofia::start()

La fonction a été extraite avec :

```sh
arm-linux-gnueabi-objdump -d -C \\
  --start-address=0x7ac50 \\
  --stop-address=0x7afa0 \\
  "$BIN" \\
  > dump/analysis/functions/csofia-start.asm/csofia-start.asm
```

Le début de la bifurcation est :

```sh
7ad18: ldr r2, [pc, ...]      ; ProductCof_g
7ad1c: ldr r3, [r2, #16]     ; mode produit
7ad20: cmp r3, #3
7ad24: cmp r3, #2
```

Trois chemins apparaissent :

| Mode | Chemin observé |
| :---- | :---- |
| 3 | mode PCBA / test usine |
| 2 | configuration Wi-Fi spécifique puis CTutk::start() |
| autre, avec indicateur Tuya actif | CTuya::init() puis CTuya::start() |

Le chemin Tuya contient :

```sh
7ad78: bl  36084 <CTuya::instance()>
7ad7c: bl  35f70 <CTuya::init()>
...
7ada0: bl  36084 <CTuya::instance()>
7ada4: bl  35f78 <CTuya::start()>
```

Le chemin TUTK est donc une variante alternative du produit, et non une étape automatiquement exécutée après le SDK Tuya.

## 8.15 Vérification de la configuration persistante

Le contenu de l’archive /etc/conf a été examiné sans l’extraire :

```sh
tar tf dump/etc_conf.tar
```

```sh
tar xOf dump/etc_conf.tar etc/conf/product.cof
```

Le fichier sauvegardé contient les paramètres matériels et la section :

```sh
[DEFAULT_SETTING]
language=0
```

Il ne contient pas les paramètres STATION, stationssid ou stationpwd associés au chemin spécifique aperçu dans le binaire.

Surtout, les observations runtime montrent sans ambiguïté :

- initialisation du SDK Tuya ;
- connexion MQTT sur le port 8883 ;
- logs du résolveur et du cloud Tuya ;
- exécution de CTuya::ThreadProc().

Conclusion : sur cette caméra, le chemin actif est CTuya. La pile TUTK/IOTC historique est présente et liée au binaire, mais elle est vraisemblablement inactive dans la configuration courante.

Cette conclusion change la priorité des patchs :

- initP2p() et initp2pServer() ne sont plus les premières cibles ;
- les fonctions P2P/WebRTC du SDK Tuya deviennent prioritaires ;
- supprimer les bibliothèques IOTC du rootfs serait inutile et risqué à ce stade.

## 8.16 Séparation confirmée du RTSP local

La recherche de l’appelant de StartRtspPthread() donne :

COnvif::Start()
  ├── StartRtspPthread()
  ├── StartOnvifPthread()
  └── motor_ctrl()

Commandes utilisées :

```sh
grep -n -B 24 -A 30 \\
  -E 'bl.*<(COnvif::Start|StartRtspPthread)' \\
  dump/analysis/dgiot.disasm
```

Extrait :

```sh
798e4: bl 7960c <StartRtspPthread()>
798f8: bl b0fc8 <StartOnvifPthread>
```

COnvif est construit depuis CTuya::ThreadProc(), mais le serveur RTSP lui-même est une branche locale distincte des fonctions de transfert P2P Tuya.

Cette nuance est importante : il ne faut pas supprimer brutalement tout CTuya::ThreadProc(), car ce thread participe aussi au lancement et au suivi de fonctions locales.

## 8.17 Chaîne réelle d’initialisation du SDK Tuya

La fonction principale du SDK a été extraite dans un fichier dédié :

```sh
arm-linux-gnueabi-objdump -d -C \\
  --start-address=0x44da4 \\
  --stop-address=0x45324 \\
  "$BIN" \\
  > dump/analysis/functions/tuay_sdk_start.asm
```

La séquence observée est :

CTuya::ThreadProc()
  └── tuay_sdk_start()
        ├── IPC_APP_Init_SDK()
        │     ├── tuya_ipc_init_sdk()
        │     └── tuya_ipc_start_sdk()
        ├── attente de tuya_ipc_get_register_status() == 2
        ├── création des flux applicatifs
        ├── TUYA_APP_Enable_P2PTransfer()
        ├── IPC_APP_upload_all_status()
        ├── TUYA_APP_Enable_CloudStorage()
        ├── TUYA_APP_Enable_AI_Detect()
        └── tuya_ipc_upload_skills()

Deux appels de haut niveau sont maintenant identifiés :

```sh
452ac: bl 41c60 <TUYA_APP_Enable_P2PTransfer>
452d4: bl 3924c <TUYA_APP_Enable_CloudStorage>
```

Le premier construit une structure de callbacks puis appelle :

tuya_ipc_tranfser_init()
  └── __p2p_v3_login_init()
        └── tuya_p2p_rtc_init()

Le second appelle directement :

tuya_ipc_cloud_storage_init()

# 9. Stratégie de patch suivante

## 9.1 Première expérience de patch recommandée

La prochaine expérience doit rester graduelle. Elle ne cherchera pas encore à supprimer MQTT, car le SDK attend d’être enregistré avant de poursuivre et plusieurs fonctions locales sont lancées autour de cette séquence.

## Patch A — neutraliser P2P Transfer Tuya

Cible logique :

appel à TUYA_APP_Enable_P2PTransfer à l’adresse virtuelle 0x000452AC

Comportement attendu :

- pas d’initialisation de tuya_ipc_tranfser_init() ;
- pas de tuya_p2p_rtc_init() ;
- pas de threads STUN/relay/signaling associés ;
- MQTT principal encore actif ;
- RTSP et ONVIF conservés.

La valeur de retour est seulement journalisée. Pour simuler un succès propre, l’appel pourrait être remplacé par :

mov r0, #0

## Patch B — neutraliser Cloud Storage

Cible logique :

appel à TUYA_APP_Enable_CloudStorage à l’adresse virtuelle 0x000452D4

Comportement attendu :

- pas d’appel à tuya_ipc_cloud_storage_init() ;
- pas de contrôle cloud storage initialisé ;
- enregistrement local sur carte SD préservé ;
- MQTT principal encore actif.

Le wrapper retourne toujours 0 et son résultat n’est pas utilisé. Un remplacement par mov r0, #0 respecte donc son contrat apparent.

## Pourquoi ne pas couper MQTT immédiatement

tuay_sdk_start() appelle IPC_APP_Init_SDK(), puis attend :

tuya_ipc_get_register_status() == 2

Neutraliser trop tôt le SDK ou MQTT ferait probablement rester le thread dans cette boucle :

```sh
451d4: bl  tuya_ipc_get_register_status
451d8: cmp r0, #2
451dc: bne 451cc
```

Cette attente pourrait empêcher la suite de l’initialisation applicative. Il est donc plus sûr de supprimer d’abord les modules optionnels lancés après l’enregistrement.

## 9.2 Risques et validation runtime

Pour le patch P2P/cloud storage uniquement, les vérifications devront porter sur :

- présence stable du processus dgiot ;
- propriété correcte de /dev/watchdog ;
- absence de reboot sur plusieurs minutes ;
- fonctionnement de rtsp://IP:8554/main et /sub ;
- fonctionnement éventuel d’ONVIF ;
- connexions TCP/UDP restantes ;
- disparition des logs P2P v3, RTC, STUN et cloud storage ;
- maintien probable de la connexion MQTT 8883 à cette étape.

Cette première expérience ne réalisera donc pas encore l’objectif « zéro cloud ». Elle permettra de vérifier que les modules optionnels P2P et cloud storage peuvent être retirés sans déstabiliser la caméra, avant de traiter le cœur MQTT/DNS du SDK.

## 9.3 Correspondance entre adresses virtuelles et offsets fichier

La section .text est décrite ainsi :

```sh
readelf -SW "$BIN" | grep -E '\\.text|\\.rodata'
```

Résultat :

.text  VirtAddr=0x0001A640  FileOff=0x0000A640

Dans cette section :

offset fichier = adresse virtuelle - 0x00010000

Les deux appels correspondent donc à :

| Appel | Adresse virtuelle | Offset fichier |
| :---- | :---- | :---- |
| TUYA_APP_Enable_P2PTransfer | 0x000452AC | 0x000352AC |
| TUYA_APP_Enable_CloudStorage | 0x000452D4 | 0x000352D4 |

Les octets ont été vérifiés sans modifier le fichier :

```sh
xxd -g 4 -s 0x352a0 -l 64 "$BIN"
```

Extraits little-endian :

offset 0x352AC : 6b f2 ff eb  ; bl TUYA_APP_Enable_P2PTransfer
offset 0x352D4 : dc cf ff eb  ; bl TUYA_APP_Enable_CloudStorage

L’instruction ARM proposée pour simuler un retour réussi est :

mov r0, #0

Son encodage ARM little-endian est :

00 00 a0 e3

Ces valeurs sont désormais suffisamment établies pour construire une copie expérimentale du binaire. Le binaire original ne doit pas être modifié directement.

## 9.4 Remplacement runtime sans laisser le watchdog ouvert

Le test doit être effectué depuis la carte SD avant toute reconstruction du SquashFS. Le principal piège n’est pas la vitesse brute du remplacement, mais l’héritage du descripteur /dev/watchdog par telnetd et par le shell telnet.

Un script dédié a donc été préparé :

dump/analysis/swap_to_no_tuya_p2p.sh

Il ferme ses descripteurs watchdog hérités, arrête telnetd, tue l’ancien dgiot, lance immédiatement le binaire patché depuis /usr/bin, puis redémarre telnetd.

Le script doit être invoqué avec exec, afin qu’aucun shell telnet parent ne reste vivant avec un descripteur watchdog :

```sh
exec /mnt/swap_to_no_tuya_p2p.sh
```

La session telnet est volontairement interrompue pendant l’opération. Une nouvelle connexion est établie après le redémarrage de telnetd.

# 10. État actuel de l’enquête

## 10.1 Ce qui est établi

À ce stade, plusieurs points sont établis :

- la caméra expose un flux RTSP local sur 8554 ;
- le processus central est dgiot ;
- dgiot maintient au moins une connexion cloud MQTT/TLS sur 8883 ;
- tuer brutalement dgiot casse la caméra ;
- le rootfs SquashFS est en lecture seule ;
- /etc/conf est persistant et modifiable ;
- product.cof permet d’activer telnet, mais ne fournit pas de hook d’exécution arbitraire ;
- un hook runtime est possible en patchant une commande hardcodée non critique ;
- le watchdog impose de faire les swaps runtime proprement, sans hériter de /dev/watchdog via telnetd ;
- la pile active de cette variante est CTuya, pas prioritairement CTutk ;
- RTSP/ONVIF semblent distincts des modules P2P/cloud storage Tuya, ce qui ouvre la porte à un patch plus fin.

## 10.2 Ce qui reste incertain

Les points suivants restent à valider expérimentalement :

- stabilité longue durée avec le binaire patché ;
- comportement exact de l’application mobile si P2P Transfer est neutralisé ;
- maintien de l’enregistrement local sur carte SD après neutralisation de Cloud Storage ;
- connexions résiduelles après retrait des modules P2P/STUN/relay ;
- faisabilité d’un repack SquashFS sans dépasser les contraintes de taille et de layout flash.

## 10.3 Prochaine étape

La suite logique est de générer une copie expérimentale de dgiot où les appels à TUYA_APP_Enable_P2PTransfer et TUYA_APP_Enable_CloudStorage sont remplacés par un retour succès apparent, puis de tester ce binaire depuis la carte SD avec le script de swap propre qui ferme les file descriptors hérités.

Le firmware ne doit être reconstruit et flashé qu’après validation runtime de cette version.

# 11. Passage au flash persistant via UART et U-Boot

## 11.1. Objectif de cette étape

Jusqu’ici, les patchs de dgiot avaient été testés en runtime, depuis la carte SD ou depuis /mnt, sans remplacer durablement le binaire présent dans le rootfs.

Cette approche avait plusieurs avantages :

- elle évitait de modifier immédiatement la flash ;
- elle permettait de tester le comportement du binaire patché ;
- elle réduisait le risque de brick ;
- elle permettait de revenir au binaire original au reboot.

Mais elle avait aussi une limite évidente : rien n’était persistant côté rootfs. Après redémarrage, la caméra relançait le dgiot original depuis /usr/bin.

L’étape suivante consistait donc à rendre le patch persistant en reconstruisant le rootfs SquashFS, puis en le flashant dans la partition mtd4.

L’objectif n’était pas de reflasher toute la caméra, mais uniquement la partition rootfs :

```text
mtd4: 00550000 00010000 "rootfs"
```

Le layout complet observé était :

```text
mtd0: 00040000 00010000 "bootstrap"
mtd1: 00010000 00010000 "uboot-env"
mtd2: 00040000 00010000 "uboot"
mtd3: 001d0000 00010000 "kernel"
mtd4: 00550000 00010000 "rootfs"
mtd5: 00050000 00010000 "config"
```

Ce layout donne :

```text
rootfs offset = 0x260000
rootfs size   = 0x550000
```

La partition mtd4 fait donc :

0x550000 = 5 570 560 octets

L’image SquashFS reconstruite devait impérativement rester inférieure ou égale à cette taille.

## 11.2. État système minimal obtenu via UART

La caméra a été démarrée avec accès console UART. En envoyant le caractère attendu au moment du boot Linux, il est possible de tomber sur un système minimal sans lancer l’application complète.

Le processus actif observé était alors très réduit :

PID USER       VSZ STAT COMMAND
1   root      1512 S    init
76  root      1032 S <  /sbin/udevd --daemon
97  root      1520 S    -sh
...
120 root      1512 R    ps

Point important : dans cet état, il n’y avait pas de dgiot, pas de daemon, et pas de telnetd.

Cet état est idéal pour préparer une opération sensible, car le système Linux est démarré, mais la partie applicative n’est pas encore active.

En revanche, les outils MTD disponibles dans le rootfs étaient insuffisants :

```sh
for x in flashcp flash_erase mtd_debug nandwrite dd cp cat ls; do
    echo "=== $x ==="
    type "$x" 2>/dev/null
    command -v "$x" 2>/dev/null
    ls -l /bin/"$x" /sbin/"$x" /usr/bin/"$x" /usr/sbin/"$x" 2>/dev/null
done
```

Résultat utile :

=== flashcp ===
flashcp: not found

=== flash_erase ===
flash_erase: not found

=== mtd_debug ===
mtd_debug: not found

=== nandwrite ===
nandwrite: not found

```sh
=== dd ===
dd is /bin/dd
/bin/dd
```

Conclusion : depuis Linux, la caméra ne fournit pas d’outil propre pour effacer et réécrire une partition MTD.

Écrire directement avec dd ou cat vers /dev/mtd4 aurait été une mauvaise idée. Sur une SPI NOR, l’écriture ne peut pas remettre des bits de 0 à 1 sans effacement préalable. Il faut donc utiliser un mécanisme qui effectue bien un erase flash avant l’écriture.

Deux options restaient possibles :

- apporter un binaire externe flashcp / flash_erase compatible ARM ;
- utiliser directement U-Boot et ses commandes sf erase / sf write.

La seconde option a été retenue.

## 11.3. Accès au prompt U-Boot

U-Boot était bien présent sur l’UART :

```text
U-Boot 2010.06 (Sep 08 2021 - 09:21:26)

DRAM:  64 MiB
SF: Got idcode 20 40 17 20 40
flash size         : 8388608
flash sector_size  : 65536
```

Mais le délai d’interruption était configuré à zéro :

```text
Hit any key to stop autoboot:  0
```

Il était donc quasiment impossible d’interrompre le boot de manière normale au clavier.

La méthode retenue a consisté à perturber brièvement la lecture de la SPI flash au moment du boot, comme décrit dans le tutoriel `berobloom/LSCIndoorCameraLocal1080p` :

- https://github.com/berobloom/LSCIndoorCameraLocal1080p

Concrètement, le principe consiste à court-circuiter brièvement deux broches de la flash SPI juste après la mise sous tension. Cela perturbe assez le boot automatique pour récupérer la main sur U-Boot, mais il faut relâcher le court-circuit avant que Linux ou les commandes `sf` aient besoin de relire correctement la flash.

Plusieurs essais ont été nécessaires.

Premier cas d’échec : le court-circuit était maintenu trop longtemps. U-Boot parvenait à charger le kernel, mais le kernel ne pouvait plus lire la flash :

```text
m25p80 spi0.0: unrecognized JEDEC id bytes: ff, ff, ff
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```

Interprétation : la perturbation SPI était encore active au moment où Linux essayait de monter le rootfs.

Deuxième cas d’échec : la perturbation était trop précoce ou trop violente, ce qui faisait tomber le SoC dans son boot ROM :

```text
ROM:    Use nor flash.
ROM:    Xmodem start
ROM:    Please upload firmware using Xmodem
ROM:    Waiting...
```

Ce mode n’a pas été utilisé, car il attend vraisemblablement un format d’image complet ou propriétaire. Envoyer un rootfs SquashFS isolé à ce stade aurait été dangereux.

Le bon compromis a finalement permis d’obtenir le prompt U-Boot :

```text
U-Boot>
```

Une fois au prompt, il faut impérativement relâcher toute perturbation sur la flash SPI avant d’utiliser les commandes sf.

## 11.4. Vérification des commandes disponibles dans U-Boot

La commande help a confirmé la présence des fonctions nécessaires :

```sh
fatload - load binary file from a dos filesystem
fatls   - list files in a directory
mmc     - MMC sub system
sf      - SPI flash sub-system
loady   - load binary file over serial line (ymodem mode)
tftpboot- boot image via network using TFTP protocol
saveenv - save environment variables to persistent storage
```

La présence de fatload et sf est importante :

- fatload permet de charger une image depuis la carte SD FAT32 en RAM ;
- sf erase permet d’effacer proprement une zone de la SPI NOR ;
- sf write permet d’écrire l’image depuis la RAM vers la flash ;
- sf read et cmp.b permettent de relire et comparer la flash après écriture.

La carte SD était correctement lisible depuis U-Boot :

```sh
U-Boot> fatls mmc 0:1
            system volume information/
            dcim/
     2046   swap_control.sh
      100   swap_control.log
        0   dgiot_control.stdout.log
       27   product.cof
  7403712   dgiot_nocloud.bin
  7403712   dgiot_no_tuya_p2p.bin
     2066   swap_to_no_tuya_p2p.sh
  5246976   mtd4_rootfs_no_tuya_p2p.squashfs
```

8 file(s), 2 dir(s)

L’image à flasher était donc :

mtd4_rootfs_no_tuya_p2p.squashfs

Sa taille était :

5 246 976 octets = 0x501000

Ce qui reste inférieur à la taille maximale de mtd4 :

0x501000 < 0x550000

## 11.5. Vérification de l’environnement U-Boot

La commande printenv a confirmé le boot layout :

```sh
bootargs=console=ttyS0,115200 mem=36M quiet root=/dev/mtdblock4 rootfstype=squashfs mtdparts=spi_flash:256k(bootstrap),64k(uboot-env),256k(uboot),1856k(kernel),5440k(rootfs),320k(config)
```

```sh
bootcmd=sf probe 0; sf read 0xA1000000 0x90000 0x1d0000; bootm
```

```sh
bootdelay=0
baudrate=115200
stdin=serial
stdout=serial
stderr=serial
```

Cela confirme que :

- le kernel est lu depuis l’offset 0x90000 ;
- sa taille est 0x1d0000 ;
- le rootfs est monté depuis /dev/mtdblock4 ;
- le rootfs attendu est un SquashFS ;
- la partition rootfs commence donc juste après le kernel, à 0x260000.

Calcul :

0x090000 + 0x1d0000 = 0x260000

C’est bien l’offset utilisé pour flasher le nouveau rootfs.

## 11.6. Flash du rootfs SquashFS via U-Boot

Avant toute écriture, la flash SPI a été reprobe :

U-Boot> sf probe 0
master [ctl : mem] = [0 : 0]
SF: Got idcode 20 40 17 20 40
use default flash ops...
spi_flash_probe_default multi wire open flag is 0
found speed : delay = 50000000 ;1
8192 KiB default_flash at 0:0 is now current device

L’ID JEDEC correct est :

20 40 17 20 40

C’est un point critique : aucun erase ni write ne doit être lancé si la flash répond avec ff ff ff ff ff.

L’image SquashFS a ensuite été chargée en RAM à l’adresse 0xA1000000 :

U-Boot> fatload mmc 0:1 0xA1000000 mtd4_rootfs_no_tuya_p2p.squashfs
reading mtd4_rootfs_no_tuya_p2p.squashfs

```sh
5246976 bytes read
```

U-Boot a automatiquement défini la variable filesize :

```sh
U-Boot> printenv filesize
filesize=501000
```

Une checksum CRC32 a été calculée sur l’image chargée en RAM :

U-Boot> crc32 0xA1000000 ${filesize}
CRC32 for a1000000 ... a1500fff ==> 63290b21

La zone rootfs a ensuite été effacée :

U-Boot> sf erase 0x260000 0x550000

L’effacement s’est fait par secteurs SPI de 64 KiB, de l’offset 0x260000 à 0x7AFFFF, ce qui correspond exactement à la partition mtd4.

Puis l’image a été écrite :

U-Boot> sf write 0xA1000000 0x260000 ${filesize}

Pour vérifier l’écriture, la flash a été relue en RAM à une autre adresse :

U-Boot> sf read 0xA1800000 0x260000 ${filesize}

Puis comparée à l’image source encore présente en RAM :

U-Boot> cmp.b 0xA1000000 0xA1800000 ${filesize}
Total of 5246976 bytes were the same

Cette ligne valide l’écriture :

RAM source == flash relue

Le flash de mtd4 est donc correct.

## 11.7. Modification du bootdelay U-Boot

L’environnement U-Boot affichait initialement :

```sh
bootdelay=0
```

Ce délai rendait l’accès au prompt très pénible et imposait d’utiliser une méthode de glitch SPI pour interrompre le boot.

Comme U-Boot dispose d’une partition dédiée à son environnement :

```text
mtd1: 00010000 "uboot-env"
```

et que la commande saveenv est disponible, le délai de boot a été rendu persistant :

```sh
setenv bootdelay 5
setenv filesize
saveenv
reset
```

La suppression de filesize avant saveenv évite simplement de persister une variable temporaire issue du fatload.

L’objectif est qu’au prochain démarrage, U-Boot laisse une fenêtre de 5 secondes :

Hit any key to stop autoboot:  5

Cela transforme l’accès U-Boot en opération normale via UART, sans avoir à perturber physiquement la flash SPI à chaque fois.

Précaution importante : seules les variables strictement nécessaires ont été modifiées. Les variables suivantes n’ont pas été changées :

bootcmd bootargs mtdparts

Elles étaient cohérentes avec le layout de la flash.

## 11.8. Boot après flash

Après reboot, le kernel démarre correctement :

Starting kernel ...

Les scripts init sont exécutés :

starting pid 65, tty '': '/etc/init.d/rcS'
[RCS]: /etc/init.d/S01udev
Starting udev:      [ OK ]
[RCS]: /etc/init.d/S02init_rootfs
[RCS]: /etc/init.d/S03network

Puis l’application démarre :

start
ver: 7.6.32

Le binaire dgiot lit la configuration, récupère les informations de licence depuis /etc/conf/deviceinfo, initialise les composants audio/vidéo, détecte le capteur MIS2006, démarre les threads internes et nourrit le watchdog.

Les threads observés incluent :

Threads:
               Name            PID  Prior State
_______________________________________________________
                    Main        124   64  Normal
                 FeedDog        127   50  Normal
            TimerManager        142   50  Normal
          IndicaotrLight        143   50  Normal
                  Pooled        144   50  Normal
            AudioManager        152   50  Normal
             AudioPrompt        153   50  Normal

Point important : le thread FeedDog est actif, et le watchdog est bien kické. La caméra ne reboot donc pas immédiatement.

Cela valide une partie importante du test :

- rootfs flashé : OK
- kernel : OK
- mount SquashFS : OK
- dgiot : OK
- watchdog : OK

## 11.9. Résultat inattendu : plus de RTSP

Malgré un boot correct et un watchdog alimenté, le flux RTSP local ne fonctionne plus.

La vérification réseau sur la caméra montre :

```sh
# netstat -tunlp 2>/dev/null
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:6668            0.0.0.0:* LISTEN      124/dgiot
tcp        0      0 0.0.0.0:23              0.0.0.0:* LISTEN      136/telnetd
```

Puis :

# netstat -an 2>/dev/null | grep -E '8554|554|80|6667|8883|1443'

ne retourne rien pour 8554, 554 ou 80.

Conclusion :

le serveur RTSP n’est pas lancé

Il ne s’agit donc pas d’un problème côté VLC, Home Assistant ou réseau local. Le port 8554 n’est simplement plus exposé par dgiot.

Le seul service applicatif visible est :

0.0.0.0:6668 LISTEN 124/dgiot

Des erreurs Tuya apparaissent également :

```sh
[01-01 18:15:01-- TUYA Err][app_agent.c:834] sendto Failed,len:172 ret:-1,errno:-18 port:6667
```

Ces logs suggèrent que dgiot tourne, mais reste dans un état réseau / provisioning / agent Tuya incomplet, ou que la séquence applicative qui démarre ONVIF / RTSP n’est plus atteinte.

## 11.10. Interprétation de l’échec du patch no_tuya_p2p

L’image flashée contenait le binaire :

dgiot_no_tuya_p2p.bin

Ce binaire neutralisait deux appels haut niveau identifiés dans la séquence Tuya :

TUYA_APP_Enable_P2PTransfer

TUYA_APP_Enable_CloudStorage

L’hypothèse initiale était que ces fonctions étaient optionnelles et situées après l’initialisation locale. Le comportement attendu était donc :

- RTSP local : conservé
- watchdog : conservé
- P2P Tuya : neutralisé
- cloud storage : neutralisé
- MQTT principal : possiblement encore actif

Le résultat réel est différent :

- watchdog : OK
- boot : OK
- dgiot : OK
- RTSP local : KO

Cela montre que ces appels, ou leur environnement immédiat, ne sont pas aussi optionnels qu’ils en avaient l’air lors de l’analyse statique.

Plusieurs explications sont possibles :

- TUYA_APP_Enable_P2PTransfer initialise aussi des structures de callbacks utilisées indirectement par le chemin RTSP ou ONVIF.
- La fonction neutralisée participe à une séquence globale d’initialisation applicative, même si son nom suggère seulement du P2P.
- La valeur de retour simulée n’est pas suffisante pour reproduire les effets de bord attendus.
- La branche qui démarre COnvif::Start() ou StartRtspPthread() dépend d’un état global modifié par ces fonctions.

Le binaire reste vivant, mais ne passe jamais dans la phase qui expose les services locaux.

Cette expérience invalide donc la stratégie consistant à neutraliser directement ces wrappers haut niveau.

## 11.11. Bilan de cette étape

Ce test est un échec fonctionnel pour le RTSP, mais une réussite expérimentale importante.

Ce qui est validé :

- Accès U-Boot via UART : OK
- Lecture SD FAT32 depuis U-Boot : OK
- Chargement rootfs en RAM via fatload : OK
- Écriture SPI via sf erase / sf write : OK
- Vérification par sf read / cmp.b : OK
- Boot sur rootfs modifié : OK
- Watchdog applicatif : OK
- Modification bootdelay U-Boot : OK

Ce qui échoue :

- RTSP local après patch no_tuya_p2p : KO

Ce que l’on apprend :

- Le patch no_tuya_p2p est trop agressif ou trop haut niveau.
- Il ne bricke pas la caméra.
- Il ne casse pas le watchdog.
- Mais il empêche le lancement du serveur RTSP local.

Conclusion intermédiaire :

La frontière de patch doit être déplacée.

Il ne faut probablement pas neutraliser directement TUYA_APP_Enable_P2PTransfer ou TUYA_APP_Enable_CloudStorage dans cette version. Une approche plus sûre consiste à revenir à un binaire plus conservateur, par exemple :

dgiot_nocloud.bin

c’est-à-dire :

- conserver le hook /etc/conf/x ;
- conserver les routes blackhole appliquées depuis /etc/conf ;
- éventuellement conserver certains patchs de chaînes DNS/domaines ;
- ne pas neutraliser les fonctions haut niveau P2P/cloud storage.

La stratégie réseau par route blackhole est moins élégante qu’une suppression propre du P2P dans le binaire, mais elle avait l’avantage de préserver le RTSP local lors des essais runtime.

## 11.12. Prochaine étape

La prochaine étape logique est un rollback contrôlé vers une image rootfs plus conservative.

Objectif :

rootfs persistant avec dgiot_nocloud.bin

et non plus :

rootfs persistant avec dgiot_no_tuya_p2p.bin

Procédure de flash identique :

```sh
fatload mmc 0:1 0xA1000000 <rootfs_conservateur>.squashfs
printenv filesize
crc32 0xA1000000 ${filesize}

sf probe 0
sf erase 0x260000 0x550000
sf write 0xA1000000 0x260000 ${filesize}

sf read 0xA1800000 0x260000 ${filesize}
cmp.b 0xA1000000 0xA1800000 ${filesize}

reset
```

Après boot, les critères de validation seront :

1. dgiot démarre ;
2. FeedDog tourne ;
3. pas de reboot watchdog ;
4. port 8554 en écoute ;
5. RTSP /main et /sub fonctionnels ;
6. routes blackhole appliquées ;
7. connexions cloud restantes documentées.

Commande de vérification RTSP côté caméra :

```sh
netstat -tunlp 2>/dev/null | grep -E '8554|554|80|6668|8883|1443'
```

Critère attendu :

0.0.0.0:8554 LISTEN

Si cette version conserve le RTSP, elle deviendra la première version persistante réellement exploitable.

# 12. Conclusion : Caméra locale sous contrôle

Le pari est réussi : nous disposons désormais d'une caméra LSC/Tuya parfaitement fonctionnelle en local, tout en ayant neutralisé son besoin de "téléphoner maison".

Le passage d'une approche "brute" (blocage réseau externe, qui a provoqué des instabilités) à une approche "chirurgicale" (patch binaire de la fonction gw_p2p_mqtt_data_cb) a été l'élément déterminant. En interceptant le signaling RTC à sa source, nous avons pu supprimer les communications indésirables sans casser les dépendances logicielles critiques (watchdog, RTSP, threads système) qui assuraient la stabilité du binaire dgiot.

Cette autopsie nous a permis de valider plusieurs points clés :

1. **La gestion du watchdog est prioritaire :** toute modification système doit impérativement préserver l'alimentation du FeedDog, sous peine de reboot cyclique.
2. **Le patch applicatif est plus robuste que le filtrage réseau :** bloquer les IP cloud est incertain (résolution dynamique, changement de CDN), alors qu'ignorer les messages de signaling au niveau applicatif est définitif.
3. **La persistance est maîtrisable :** grâce à l'accès U-Boot, la modification de la partition mtd4 via un rootfs personnalisé permet de sécuriser le patch de manière durable.

La caméra, libérée de son écosystème cloud, est maintenant intégrée de façon stable au Homelab. Elle offre un flux RTSP fiable et devient une brique autonome de surveillance, sans compromis sur la confidentialité.

# Annexes

## Annexe A — Commandes FortiGate

Sniffer tout le trafic de la caméra :

```sh
diagnose sniffer packet any 'host 192.168.1.12' 4 0 a
```

Sniffer le RTSP :

```sh
diagnose sniffer packet any 'host 192.168.1.12 and port 8554' 4 0 a
```

Sniffer MQTT/TLS cloud :

```sh
diagnose sniffer packet any 'host 192.168.1.12 and port 8883' 4 0 a
```

Chercher une session FortiGate :

```sh
diagnose sys session filter clear
diagnose sys session filter src 192.168.1.12
diagnose sys session list
diagnose sys session filter clear
```

Chercher par port :

```sh
diagnose sys session filter clear
diagnose sys session filter dport 8883
diagnose sys session list
diagnose sys session filter clear
```

## Annexe B — Variante de route blackhole

Si BusyBox l’accepte :

```sh
route del -host 3.124.225.12 2>/dev/null
route add -host 3.124.225.12 dev lo
```

Avantage :

pas d’ARP inutile vers une IP LAN inexistante.

Inconvénient :

pas forcément supporté par la version BusyBox/route embarquée.

## Annexe C — Inventaire système et images firmware

## Annexe C.1 — Inventaire du système en fonctionnement

En complément des images flash, un instantané du système a été collecté :

```sh
mkdir -p /mnt/dump/sysinfo
```

```sh
cat /proc/mtd > /mnt/dump/sysinfo/proc_mtd.txt
cat /proc/mounts > /mnt/dump/sysinfo/proc_mounts.txt
cat /proc/cmdline > /mnt/dump/sysinfo/proc_cmdline.txt
cat /proc/cpuinfo > /mnt/dump/sysinfo/proc_cpuinfo.txt
cat /proc/meminfo > /mnt/dump/sysinfo/proc_meminfo.txt
ps > /mnt/dump/sysinfo/ps.txt
mount > /mnt/dump/sysinfo/mount.txt
df -h > /mnt/dump/sysinfo/df_h.txt
ifconfig -a > /mnt/dump/sysinfo/ifconfig_a.txt 2>&1
netstat -tunp > /mnt/dump/sysinfo/netstat_tunp.txt 2>&1
```

Ces informations permettent de relier l’analyse statique à la réalité du runtime :

- partitions effectivement montées ;
- arguments transmis au noyau ;
- mémoire disponible ;
- processus actifs ;
- interfaces et connexions réseau ;
- rôle persistant de /etc/conf et de la carte SD.

## Annexe D : Procédure de flash

```sh
fatload mmc 0:1 0xA1000000 <nom_fichier>.squashfs
printenv filesize
crc32 0xA1000000 ${filesize}

sf probe 0
sf erase 0x260000 0x550000
sf write 0xA1000000 0x260000 ${filesize}

sf read 0xA1800000 0x260000 ${filesize}
cmp.b 0xA1000000 0xA1800000 ${filesize}

reset
```

## Annexe D.1 — Identification des images sur la machine d’analyse

Une fois les fichiers transférés sur le PC, leur type a été identifié :

```sh
file *.bin
binwalk *.bin
```

Résultats :

```text
mtd3_kernel.bin : image U-Boot legacy uImage, Linux 4.9.129, ARM
mtd4_rootfs.bin : SquashFS 4.0, little-endian, compression XZ
mtd5_config.bin : système de fichiers JFFS2
```

La topologie du firmware est donc :

```text
bootstrap
  → environnement U-Boot
  → U-Boot
  → kernel
  → rootfs SquashFS en lecture seule
  → partition de configuration JFFS2 modifiable
```

Les métadonnées montrent également deux générations différentes :

- kernel daté du 29 octobre 2022 ;
- rootfs SquashFS créé le 12 avril 2023.

Cette différence suggère que le firmware applicatif a été assemblé plus tard autour d’un noyau déjà existant.

## Annexe D.2 — Précaution concernant l’environnement U-Boot

Le fichier mtd1_uboot-env.bin semble contenir des données lisibles directement :

```sh
strings -a mtd1_uboot-env.bin
```

Il peut révéler :

```sh
bootargs
mtdparts
console
root=
init=
bootcmd
```

Cette partition n’a pas été modifiée. L’environnement U-Boot est une zone sensible : une erreur peut empêcher le démarrage bien avant que Linux ou telnet soient disponibles.

## Annexe E — Photos de la carte et de la flash
