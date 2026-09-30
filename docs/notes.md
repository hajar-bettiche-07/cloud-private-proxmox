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


## Problèmes rencontrés et solutions

### 1. Installation Debian via ISO instable
- Symptôme : console noVNC qui perd des touches (« root » devenait « r »), VM qui s'éteint, boot bloqué après l'installation
- Diagnostic : RAM et CPU de pve-node1 augmentés (4 → 6 Go, 4 CPU), disque vérifié (SSD) : aucune amélioration. Cause exacte non identifiée (3 couches de virtualisation imbriquées : Windows → VirtualBox → Proxmox → VM)
- Solution : contourner le problème avec une image cloud + cloud-init, sans aucune interaction en console

### 2. La VM n'obtient pas d'adresse IPv4 (DHCP)
- Symptôme : impossible de trouver l'IP de la VM (`qm guest cmd` échoue, pas d'agent QEMU dans l'image)
- Diagnostic : `ip neigh` sur pve1 montre la VM sur vmbr0 avec une adresse IPv6 locale seulement, donc aucun serveur DHCP ne répond
- Solution : IP fixe via cloud-init : `qm set 102 --ipconfig0 ip=10.0.2.50/24,gw=10.0.2.2 --nameserver 8.8.8.8`, puis `qm stop 102` et `qm start 102`
- Vérification : `tcpdump -i tap102i0 -nn` montre la VM qui utilise 10.0.2.50 et joint un serveur NTP

### 3. SSH : « Permission denied (publickey) »
- Cause : les images cloud n'acceptent que les clés SSH ; le mot de passe cloud-init sert à la console uniquement
- Solution : génération d'une clé (`ssh-keygen -t ed25519`) sur pve1, puis `qm set 102 --sshkeys ~/.ssh/id_ed25519.pub` et redémarrage de la VM

### 4. SSH : « REMOTE HOST IDENTIFICATION HAS CHANGED »
- Cause : cloud-init régénère les clés d'hôte de la VM au redémarrage
- Solution : `ssh-keygen -R 10.0.2.50`, puis reconnexion (attendu ici, pas une attaque)

### 5. Espace de stockage limité
- Symptôme : avertissement LVM-thin (moins de 8 Go libres dans le pool)
- Solution : suppression des VMs de test inutiles (100 et 101) avec `--purge --destroy-unreferenced-disks`