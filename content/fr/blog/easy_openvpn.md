---
author: "Alexis LEBEL"
title: "Installer facilement un serveur OpenVPN"
date: 2022-05-23
tags: ["openvpn", "serveur", "installation", "configuration", "serveur", "vpn"]
thumbnail: /images/2022/05/openvpn.webp
---

Un VPN est un tunnel complètement opaque de l'extérieur qui permet de joindre deux réseaux physiques distincts.
Les applications techniques d'un VPN sont multiples :

- Accès à des ressources d'entreprise sur un site distant
- Sécurisation des échanges d'information entre deux réseaux
- Jeu en ligne : permet de sécuriser un serveur de jeu
- Anonymat et confidentialité : permet de protéger les données personnelles en délocalisant les utilisateurs
- Télétravail : Accès aux ressources de l'entreprise depuis un domicile
- ...

En temps normal, mettre en place un serveur VPN nécéssite des connaissances pointues en réseau et en système. Cependant, il existe des outils pour simplifier ce travail.

# Prérequis

Pour déployer un serveur VPN, vous allez avoir besoin d'un serveur. Cela peut être un serveur virtuel (VPS), une instance cloud (GCP, AWS, Azure) ou même un simple Raspberry Pi.

Ensuite, il faut s'assurer que le port 1194/UDP soit bien ouvert sur le réseau qui accueille la machine. Pour cela, il faut se rendre dans les paramètres de votre hébergeur, ou de votre box.

Cette machine doit tourner sous Ubuntu, Debian, AlmaLinux, Rocky Linux, CentOS ou Fedora.

# Installation

Tout d'abord, il faut télécharger le script d'installation de OpenVPN.

```bash
wget https://git.io/vpn -O openvpn-install.sh
```

Si cette commande renvoie une erreur, il faut effectuer la commande :
    
```bash
sudo apt install wget # Ubuntu
sudo yum install wget # CentOS, Rocky Linux
sudo dnf install wget # Fedora
```

Ensuite, il faut rendre le script exécutable :

```bash
chmod +x openvpn-install.sh
```

Puis le lancer :

```bash
sudo ./openvpn-install.sh
```

L'installation devrait se démarrer comme suit :

```sh
Welcome to this OpenVPN road warrior installer!

This server is behind NAT. What is the public IPv4 address or hostname?
Public IPv4 address / hostname [xxx.xxx.xxx.xxx]: # IP Publique du serveur, déterminée automatiquement

Which protocol should OpenVPN use?
   1) UDP (recommended)
   2) TCP
Protocol [1]: # Protocole utilisé par OpenVPN, généralement UDP (par défaut)

What port should OpenVPN listen to?
Port [1194]: # Port utilisé par OpenVPN, généralement 1194 (par défaut)

Select a DNS server for the clients:
   1) Current system resolvers
   2) Google
   3) 1.1.1.1
   4) OpenDNS
   5) Quad9
   6) AdGuard
DNS server [1]: # Serveur DNS utilisé par les clients, généralement le DNS du système (par défaut)

Enter a name for the first client:
Name [client]: personne_1 # Nom d'utilisateur de la première personne qui utilisera OpenVPN

OpenVPN installation is ready to begin.
Press any key to continue...
```

A partir de là, OpenVPN est installé et le serveur est démarré. Le serveur est accessible depuis le port 1194/UDP.

Le premier client est créé automatiquement. Il se trouve dans le répertoire /root.

# Configuration des clients

Pour créer de nouveaux clients, il faut exécuter le script d'installation.

```bash
sudo ./openvpn-install.sh
```

Cela devrait donner l'interface suivante :
    
```sh
OpenVPN is already installed.

Select an option:
   1) Add a new client
   2) Revoke an existing client
   3) Remove OpenVPN
   4) Exit
Option: 1 

Provide a name for the client:
Name: client2
```

# Désinstallation

Pour désinstaller OpenVPN, il faut exécuter le script d'installation.

```bash
sudo ./openvpn-install.sh
```

Cela devrait donner l'interface suivante :
    
```sh
OpenVPN is already installed.

Select an option:
   1) Add a new client
   2) Revoke an existing client
   3) Remove OpenVPN
   4) Exit
Option: 3 
```

Et voilà, OpenVPN est désinstallé.

# Conclusion

Maintenant, vous savez déployer un serveur VPN très rapidement, cela peut vous permettre de vous adapter rapidement à des problématiques d'accès distant.
Aujourd'hui, avec le télétravail, cela peut s'avérer indispensable ;) !