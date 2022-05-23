---
author: "Alexis LEBEL"
title: "Easily install an OpenVPN server"
date: 2022-05-23
tags: ["openvpn", "server", "installation", "configuration", "vpn"]
thumbnail: /images/2022/05/openvpn.webp
---

A VPN is a tunnel that is completely opaque to the outside world and allows two distinct physical networks to be joined.
The technical applications of a VPN are multiple:

- Access to corporate resources on a remote site
- Securing information exchanges between two networks
- Online gaming: to secure a game server
- Anonymity and confidentiality: protect personal data by relocating users
- Telecommuting : access to company resources from home
- ...

Normally, setting up a VPN server requires advanced network and system knowledge. However, there are tools to simplify this work.

# Prerequisites

To deploy a VPN server, you will need a server. This can be a virtual server (VPS), a cloud instance (GCP, AWS, Azure) or even a simple Raspberry Pi.

Next, you need to make sure that the 1194/UDP port is open on the network that hosts the machine. To do this, you need to go to the settings of your host or your box.

This machine must run under Ubuntu, Debian, AlmaLinux, Rocky Linux, CentOS or Fedora.

# Installation

First of all, you need to download the OpenVPN installation script.

```bash
wget https://git.io/vpn -O openvpn-install.sh
```

If this command returns an error, you need to run the command :
    
```bash
sudo apt install wget # Ubuntu
sudo yum install wget # CentOS, Rocky Linux
sudo dnf install wget # Fedora
```

Next, you need to make the script executable:

```bash
chmod +x openvpn-install.sh
```

Then run it:

```bash
sudo ./openvpn-install.sh
```

The installation should start as follows:

```sh
Welcome to this OpenVPN road warrior installer!

This server is behind NAT. What is the public IPv4 address or hostname?
Public IPv4 address / hostname [xxx.xxx.xxx.xxx]: # Public IP of the server, determined automatically

Which protocol should OpenVPN use?
   1) UDP (recommended)
   2) TCP
Protocol [1]: # Protocol used by OpenVPN, usually UDP (default)

What port should OpenVPN listen to?
Port [1194]: # Port used by OpenVPN, usually 1194 (default)

Select a DNS server for the clients:
   1) Current system resolvers
   2) Google
   3) 1.1.1.1
   4) OpenDNS
   5) Quad9
   6) AdGuard
DNS server [1]: # DNS server used by the clients, usually the system DNS (default)

Enter a name for the first client:
Name [client]: person_1 # User name of the first person who will use OpenVPN

OpenVPN installation is ready to begin.
Press any key to continue...
```

Now OpenVPN is installed and the server is started. The server is accessible from port 1194/UDP.

The first client is created automatically. It is located in the /root directory.

# Client configuration

To create new clients, you have to run the installation script.

```bash
sudo ./openvpn-install.sh
```

This should give the following interface:
    
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

# Uninstall

To uninstall OpenVPN, you need to run the installation script.

```bash
sudo ./openvpn-install.sh
```

This should give the following interface:
    
```sh
OpenVPN is already installed.

Select an option:
   1) Add a new client
   2) Revoke an existing client
   3) Remove OpenVPN
   4) Exit
Option: 3 
```

And that's it, OpenVPN is uninstalled.

# Conclusion

Now, you know how to deploy a VPN server very quickly, it can allow you to adapt quickly to remote access problems.
Nowadays, with remote working, it can be essential ;) !