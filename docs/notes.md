# Notes projet Proxmox

## Environnement
- ASUS Vivobook X1502VA, i7-13620H, 16 Go RAM, Windows 11
- VirtualBox : VM pve-node1, 2 vCPU, 4096 Mo RAM, disque 60 Go

## Installation
- Proxmox VE 9.2, filesystem ext4, disque 60 Go
- Hostname : pve1.local
- IP : 10.0.2.15/24, gateway 10.0.2.2, DNS 8.8.8.8
ok
## Accès web (NAT + port forwarding)
- Port forwarding VirtualBox : hôte 8006 → invité 10.0.2.15:8006
- Accès : https://localhost:8006


## Dépôts APT
- Désactivé pve-enterprise et ceph enterprise (nécessitent souscription payante)
- Activé le dépôt gratuit pve-no-subscription
- apt-get update / upgrade OK, reboot effectué


## Validation nested KVM
- Test : VM Alpine Linux (1 vCPU, 512 Mo RAM) créée dans Proxmox
- Problème initial : Hyper-V/VBS Windows bloquait le VT-x malgré nested activé dans VirtualBox
- Fix : désactivation Plateforme machine virtuelle + WSL + bcdedit hypervisorlaunchtype off + reboot complet
- Résultat : VM Alpine bootée avec succès → nested KVM fonctionnel