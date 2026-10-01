# Notes projet Proxmox

## Environnement
- ASUS Vivobook X1502VA, i7-13620H, 16 Go RAM, Windows 11
- VirtualBox : VM pve-node1, 4 vCPU, 6144 Mo RAM, disque 60 Go

## Installation
- Proxmox VE 9.2, filesystem ext4, disque 60 Go
- Hostname : pve1.local
- IP : 10.0.2.15/24, gateway 10.0.2.2, DNS 8.8.8.8

## Accès web (NAT + port forwarding)
- Port forwarding VirtualBox : hôte 8006 → invité 10.0.2.15:8006
- Accès : https://localhost:8006

## Dépôts APT
- Désactivé pve-enterprise et ceph enterprise (nécessitent une souscription payante)
- Activé le dépôt gratuit pve-no-subscription
- apt-get update / upgrade OK, reboot effectué

## Validation nested KVM
- Test : VM Alpine Linux (1 vCPU, 512 Mo RAM) créée dans Proxmox
- Problème initial : la VM refusait de démarrer (« KVM virtualisation configuration is inaccessible »)
- Diagnostic : `cat /proc/cpuinfo | grep vmx` sur pve1 renvoyait un résultat vide ; `msinfo32` sous Windows affichait « Sécurité basée sur la virtualisation : En cours d'exécution »
- Cause : Hyper-V/VBS de Windows gardait le contrôle de VT-x malgré l'option nested activée dans VirtualBox
- Fix : désactivation de « Plateforme d'ordinateur virtuel » et du sous-système WSL, `bcdedit /set hypervisorlaunchtype off`, redémarrage complet du PC
- Résultat : `grep vmx` affiche des flags, la VM Alpine démarre → nested KVM fonctionnel

## Création de VM avec une image cloud
- Problème : l'installation Debian depuis l'ISO via la console noVNC était instable
- Solution : utiliser l'image cloud officielle Debian 12 (qcow2), déjà installée, importée directement dans Proxmox
- Commandes (Shell de pve1) :

```
wget --no-check-certificate https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2
qm create 102 --name serveur-dev-v2 --memory 1024 --cores 1 --net0 virtio,bridge=vmbr0
qm importdisk 102 debian-12-generic-amd64.qcow2 local-lvm
qm set 102 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-102-disk-0
qm set 102 --boot order=scsi0
qm set 102 --ide2 local-lvm:cloudinit
qm resize 102 scsi0 +12G
```

- cloud-init configure l'utilisateur, le réseau et la clé SSH au premier démarrage, sans console
- Cette VM 102 est devenue plus tard le template `template-debian12`

## Problèmes rencontrés et solutions

### 1. Installation Debian via ISO instable
- Symptôme : console noVNC qui perd des touches (« root » devenait « r »), VM qui s'éteint, boot bloqué après l'installation
- Diagnostic : RAM de pve-node1 portée de 4 à 6 Go et CPU de 2 à 4, disque vérifié (SSD) : aucune amélioration
- Cause probable : instabilité de la console noVNC en virtualisation imbriquée (Windows → VirtualBox → Proxmox → VM), hypothèse non prouvée
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

### 6. Clé SSH générée au mauvais endroit
- Symptôme : `ssh-keygen` a créé les fichiers `yes` et `yes.pub` dans le dossier personnel, au lieu de `~/.ssh/id_ed25519`
- Cause : à la question « Enter file in which to save the key », j'ai répondu `yes` (réflexe de la question d'empreinte SSH)
- Solution : suppression des mauvais fichiers, puis nouvelle génération en appuyant seulement sur Entrée, et vérification avec `ls ~/.ssh/`

### 7. « No route to host » sur un clone fraîchement démarré
- Symptôme : `ssh` vers le clone échoue juste après `qm start`
- Cause : cloud-init est relancé au premier démarrage du clone, ce qui prend 2 à 3 minutes dans cet environnement imbriqué
- Diagnostic : `qm status` (VM en marche), puis `ping` qui répond une fois le démarrage terminé
- Solution : attendre la fin du premier boot, sans rien modifier

## Agent QEMU
- Installation de `qemu-guest-agent` dans la VM, puis vérification depuis pve1 avec `qm guest cmd <id> network-get-interfaces`
- Résultat : l'IP de la VM est visible sans SSH, ce qui règle le problème du début (« No QEMU guest agent configured »)
- Le message de `systemctl enable` sur cette unité n'est pas une erreur : le service est démarré automatiquement par udev

## Template et clonage
- VM 102 nettoyée avant conversion : `apt clean`, `cloud-init clean --logs`, `/etc/machine-id` vidé, historique effacé
- 102 convertie en template et renommée `template-debian12`
- Clones liés (linked clones), créés avec `qm clone 102 <id> --name <nom>` : ils partagent le disque du template et ne stockent que leurs différences
- Vérification avec `lvs` : la colonne `Origin` des disques `vm-103-disk-0`, `vm-104-disk-0` et `vm-105-disk-0` affiche `base-102-disk-0`
- Résultat : le pool `data` reste à environ 12 % d'utilisation malgré trois clones de 15 Go

| N° | Nom | Rôle | IP |
|----|-----|------|----|
| 103 | serveur-dev | Environnement de développement | 10.0.2.51 |
| 104 | serveur-prod | Environnement de production | 10.0.2.52 |
| 105 | controleur-ansible | Contrôleur Ansible | 10.0.2.53 |

- Chaque clone reçoit son hostname, son IP fixe et sa clé SSH via cloud-init (`qm set <id> --ipconfig0 ip=...,gw=...`)
- Le template ne démarre jamais : c'est un modèle
- Ne pas supprimer le template tant que des clones liés existent, sinon ils cessent de fonctionner

## Accès SSH du contrôleur vers dev et prod
- `controleur-ansible` possède sa propre paire de clés ed25519
- Sa clé publique est ajoutée au fichier `authorized_keys` de dev et prod (avec `>>` pour ne pas écraser la clé existante de pve1)
- Test depuis le contrôleur : `ssh admin@10.0.2.51 hostname` renvoie `serveur-dev`, et `ssh admin@10.0.2.52 hostname` renvoie `serveur-prod`, sans mot de passe
- C'est ce qu'utilisera Ansible pour se connecter