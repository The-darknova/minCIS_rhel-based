# RHEL based

# **Guide de Durcissement (Hardening) pour Systèmes basés sur RHEL**

Ce document est un guide opérationnel destiné aux administrateurs système pour sécuriser les serveurs basés sur la distribution RedHat. Il suit le principe du moindre privilège, de la réduction de la surface d'attaque et de la défense en profondeur.

ATTENTION : L'application de ces paramètres doit toujours être testée dans un environnement de pré-production avant toute mise en production. Certaines restrictions peuvent altérer le fonctionnement de vos services légitimes.

Il est important de savoir que cette procedure n’est pas valable pour des systemes en End-Of-Life. 

## **Phase 1: Protection des Systèmes de Fichiers**

L'objectif est d'empêcher le montage de systèmes de fichiers obsolètes ou non sécurisés et de restreindre les permissions sur partitions temporaires ou utilisateurs.

### **1.1 Désactivation des modules de systèmes de fichiers non utilisés**

Ces systèmes de fichiers (souvent liés à d'anciens supports ou périphériques) augmentent la surface d'attaque du noyau.

**Action :** Blacklister les modules.

```bash
sudo bash -c 'cat <<EOF > /etc/modprobe.d/hardening-fs.conf
 install cramfs /bin/false
 blacklist cramfs
 install freevxfs /bin/false
 blacklist freevxfs
 install jffs2 /bin/false
 blacklist jffs2
 install hfs /bin/false
 blacklist hfs
 install hfsplus /bin/false
 blacklist hfsplus
 install squashfs /bin/false
 blacklist squashfs
 install udf /bin/false
 blacklist udf
 install usb-storage /bin/false
 blacklist usb-storage
 install firewire-core /bin/false
 blacklist firewire-core
 install afs /bin/false
 blacklist afs
 EOF'
```

**Action :** Décharger les modules s'ils sont actuellement en mémoire.

```bash
sudo bash -c \
 'for mod in cramfs freevxfs jffs2 hfs hfsplus udf usb-storage firewire-core afs; do
     modprobe -r $mod 2>/dev/null && rmmod $mod 2>/dev/null
 done'
```

### **1.2 Sécurisation des partitions**

**Attention: Cette partie est uniquement applicable si la partition existe.**

Il faut seulement ajouter les parametres correspondantes en fin de lignes existante. Ne pas ajouter de ligne dans le fichier fstab si elle est absente.

Limiter les droits d'exécution (noexec), la création de périphériques (nodev) et les permissions SUID (nosuid) sur les dossiers sensibles.

**Action :** Éditez le fichier /etc/fstab et ajoutez les options requises :

- /tmp : nodev, nosuid, noexec
- /var/tmp : nodev, nosuid, noexec
- /var : nodev, nosuid
- /dev/shm : nodev, nosuid, noexec
- /home : nodev, nosuid
- /var/log : nodev, nosuid, noexec
- /var/log/audit : nodev, nosuid, noexec**Action :** Remonter les partitions à chaud.

```bash
sudo mount -o remount /tmp
 sudo mount -o remount /var/tmp
 sudo mount -o remount /dev/shm
 sudo mount -o remount /home
 sudo mount -o remount /var/log/audit
```

### **Phase 2: Gestion Sécurisée des Paquets**

### **2.1 S’assurer que les repos utilisés sont correctes**

```bash
cat /etc/yum.repos.d/*
```

### **2.2 Limitation des dépendances faibles**

Pour éviter d'installer des paquets inutiles (surface d'attaque supplémentaire), désactivez les "Recommends" et "Suggests" d'APT.

```bash
# Vérifie si la directive existe pour la modifier, sinon l'insère sous la section [main]
if grep -q "^install_weak_deps" /etc/dnf/dnf.conf; then
    sudo sed -i 's/^install_weak_deps=.*/install_weak_deps=False/' /etc/dnf/dnf.conf
else
    sudo sed -i '/^\[main\]/a install_weak_deps=False' /etc/dnf/dnf.conf
fi
```

### **2.3 Sécurisation des dépôts DNF**

Restreindre la modification des fichiers de dépôts à l'utilisateur root.

```bash
sudo chown -R root:root /etc/dnf
 sudo find /etc/dnf -type d -exec chmod 755 {} +
 sudo find /etc/dnf -type f -exec chmod 644 {} +
```

### **2.4 Mises à jour du système**

S'assurer que le système dispose des derniers correctifs de sécurité.

```bash
sudo dnf update
# Vérifier si un redémarrage complet est nécessaire
sudo dnf needs-restarting -r || sudo reboot
```

### **2.5 Synchronisation de l'Heure (Chrony)**

Une synchronisation précise de l'heure est essentielle pour la corrélation des journaux système et d'audit.

**Action :** Installer et configurer Chrony.

```bash
sudo dnf install -y chrony
sudo bash -c 'cat <<EOF > /etc/chrony.conf
pool pool.ntp.org iburst
driftfile /var/lib/chrony/drift
log tracking measurements statistics
logdir /var/log/chrony
maxupdateskew 100.0
rtcsync
makestep 1 3
EOF'
sudo systemctl enable --now chronyd
```

### **Phase 3: Contrôle d'Accès Obligatoire (SELinux)**

SELinux confine les programmes à un ensemble limité de ressources, atténuant ainsi les attaques de type 0-day.

### **3.1 Protection du Bootloader contre la désactivation de SELinux**

La compromission initiale d'un système durci implique fréquemment la tentative de désactiver SELinux au démarrage. Les paramètres passés par le gestionnaire d'amorçage au noyau dominent les configurations locales. Historiquement, l'ajout de selinux=0 ou enforcing=0 à la ligne de commande GRUB permettait de redémarrer le système dans un état vulnérable.

Les politiques CIS RHEL 8, 9 et 10 exigent que ces paramètres soient traqués et éradiqués de l'intégralité des entrées de l'amorceur. Le manipulateur officiel du noyau sur Red Hat n'est pas la simple régénération avec grub2-mkconfig, mais l'utilitaire bas niveau grubby.

L'audit de conformité s'effectue via la scrutation de tous les noyaux installés :

```bash
sudo grubby --info=ALL | grep -Po '(selinux|enforcing)=0\b'
```

Si ces variables sont découvertes, l'administrateur doit les purger de manière universelle:

```bash
sudo grubby --update-kernel ALL --remove-args "selinux=0 enforcing=0"
```

### **3.2 Configuration Globale et Politique**

L'état nominal et inaltérable de SELinux doit être configuré dans le fichier racine /etc/selinux/config. Il exige deux déclarations strictes :

- `SELINUX=enforcing` : Assure que le noyau bloque activement les interactions illégitimes et génère une entrée d'audit (Access Vector Cache - AVC denial). Le mode *permissive* est formellement proscrit en production, car il se contente de journaliser l'infraction sans l'endiguer.
- `SELINUXTYPE=targeted` : Applique la politique ciblée, qui sandbox les démons systèmes et réseaux critiques tout en laissant les processus interactifs des utilisateurs standards non confinés (bien qu'ils demeurent soumis aux restrictions classiques de l'OS). C'est l'équilibre parfait entre haute sécurité et opérabilité.

### **3.3 Traitement des anomalies de confinement et des paquets à risques**

L'audit d'un système Red Hat révèle parfois des démons (daemons) qui, bien que SELinux soit actif, fonctionnent dans un état non confiné (`unconfined_service_t` ou `unconfined_t`). Cela se produit fréquemment avec des logiciels tiers ou des agents de sauvegarde propriétaires dépourvus de règles SELinux intégrées. Il est exigé de traquer ces processus (via la commande `ps -eZ | grep unconfined_service_t`) et de développer des modules de politique locaux via les outils `checkmodule` et `semodule_package` pour circonscrire précisément leurs droits (fichiers accessibles, ports écoutés). L'exécution ininterrompue d'un service non confiné annule la protection MAC pour ce vecteur spécifique.

Parallèlement, la présence de deux paquets utilitaires SELinux pose un risque de sécurité systémique et leur désinstallation est une directive impérative:

L'éradication s'effectue via la commande de gestion des paquets :

| **Paquet Logiciel** | **Fonction Théorique** | **Risque de Sécurité Imposant la Désinstallation** |
| --- | --- | --- |
| mcstrans | (MCS Translation Service) Traduit les catégories cryptiques de sécurité SELinux (ex. c0, c1023) en labels textuels compréhensibles par l'humain. | Introduit un démon réseau supplémentaire (mcstransd) dont la surface d'attaque n'est d'aucune utilité opérationnelle sur un serveur dépourvu de politique Multi-Level Security (MLS) complexe. |
| setroubleshoot | Capture les refus AVC générés par le noyau et génère des alertes système via D-Bus incluant des suggestions humaines pour corriger l'erreur (via la commande sealert). | Ce service historique possède de redoutables antécédents de vulnérabilités (Exécution de Code à Distance et Élévation de Privilèges). En tentant d'analyser les chaînes de caractères bloquées par le noyau, il parse parfois des données forgées par l'attaquant, permettant de contourner les protections. Son utilisation est bannie en production. |

```bash
sudo dnf remove mcstrans setroubleshoot -y
```

## **Phase 4: Sécurisation du Bootloader**

Empêcher un utilisateur local non privilégié de modifier les paramètres de démarrage (comme le passage en single-user mode).

```bash
sudo chown root:root /boot/grub/grub.cfg
sudo chmod u-x,go-rwx /boot/grub/grub.cfg
sudo chown root:root /boot/grub2/user.cfg
sudo chmod 0600 /boot/grub2/user.cfg
sudo chown root:root /boot/grub2/grub.cfg
sudo chmod 0600 /boot/grub2/grub.cfg
```

## **Phase 5: Durcissement du Noyau et des Processus**

### **5.1 Optimisation des paramètres Sysctl**

Ce fichier configure la randomisation de l'espace d'adressage (ASLR), restreint l'accès aux journaux du noyau, et protège les liens symboliques/physiques.

```bash
sudo bash -c 'cat <<EOF > /etc/sysctl.d/60-process-hardening.conf
 kernel.randomize_va_space = 2
 kernel.yama.ptrace_scope = 1
 fs.suid_dumpable = 0
 kernel.dmesg_restrict = 1
 kernel.kptr_restrict = 2
 fs.protected_hardlinks = 1
 fs.protected_symlinks = 1
 EOF'
 sudo sysctl --system
```

### **5.2 Restriction des Core Dumps et Outils de Débogage**

Empêcher le système de vider la RAM sur le disque lors d'un crash (pourrait contenir des mots de passe).

```bash
printf '%s\n%s\n%s\n' '[Coredump]' 'Storage=none' 'ProcessSizeMax=0' | \
 sudo tee -a /etc/systemd/coredump.conf.d/60-coredump.conf
```

### **5.3 Installer auditd**

```bash
sudo dnf install auditd audispd-plugins
sudo systemctl enable auditd || true
sudo grubby --update-kernel ALL --args 'audit=1'
```

### **5.4 Déploiement des règles auditd**

Déployer des règles spécifiques (CIS) pour auditd afin d'assurer une surveillance complète.

**Copiez les règles suivantes dans /etc/audit/rules.d/60-cis-auditd.rules**

```bash
## =========================================================
## CIS RHEL Benchmark - Auditd Rules
## =========================================================

## 1. Delete all existing rules to start with a clean slate
-D

## 2. Set buffer size (increase if your system handles high traffic)
-b 8192

## 3. Handle failure modes (1=printk, 2=panic, 0=silent)
-f 1

## =========================================================
## Time and Date Modification
## =========================================================
-a always,exit -F arch=b64 -S adjtimex,settimeofday,clock_settime -k time-change
-a always,exit -F arch=b32 -S adjtimex,settimeofday,clock_settime -k time-change
-w /etc/localtime -p wa -k time-change

## =========================================================
## System Locale and Network Environment (RHEL paths)
## =========================================================
-a always,exit -F arch=b64 -S sethostname,setdomainname -k system-locale
-a always,exit -F arch=b32 -S sethostname,setdomainname -k system-locale
-w /etc/issue -p wa -k system-locale
-w /etc/issue.net -p wa -k system-locale
-w /etc/hosts -p wa -k system-locale
-w /etc/hostname -p wa -k system-locale
-w /etc/sysconfig/network -p wa -k system-locale
-w /etc/sysconfig/network-scripts/ -p wa -k system-locale

## =========================================================
## MAC Policy (SELinux - RHEL Specific)
## =========================================================
-w /etc/selinux/ -p wa -k MAC-policy
-w /usr/share/selinux/ -p wa -k MAC-policy

## =========================================================
## User Emulation
## =========================================================
-a always,exit -F arch=b64 -S execve -C uid!=euid -F euid=0 -k user_emulation
-a always,exit -F arch=b32 -S execve -C uid!=euid -F euid=0 -k user_emulation

## =========================================================
## Logins and Logouts (faillock for RHEL 8+)
## =========================================================
-w /var/log/lastlog -p wa -k logins
-w /var/log/faillog -p wa -k logins
-w /var/run/faillock/ -p wa -k logins

## =========================================================
## Session Initiation Information
## =========================================================
-w /var/run/utmp -p wa -k session
-w /var/log/wtmp -p wa -k session
-w /var/log/btmp -p wa -k session

## =========================================================
## Identity and User Information Modifications
## =========================================================
-w /etc/group -p wa -k identity
-w /etc/passwd -p wa -k identity
-w /etc/gshadow -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/security/opasswd -p wa -k identity

## =========================================================
## Modifications to System Administration Scope (sudoers)
## =========================================================
-w /etc/sudoers -p wa -k scope
-w /etc/sudoers.d/ -p wa -k scope

## =========================================================
## System Administrator Actions (sudo log)
## =========================================================
-w /var/log/sudo.log -p wa -k actions

## =========================================================
## Discretionary Access Control (DAC) Permissions
## =========================================================
-a always,exit -F arch=b64 -S chmod,fchmod,fchmodat -F auid>=1000 -F auid!=unset -k perm_mod
-a always,exit -F arch=b32 -S chmod,fchmod,fchmodat -F auid>=1000 -F auid!=unset -k perm_mod
-a always,exit -F arch=b64 -S chown,fchown,fchownat,lchown -F auid>=1000 -F auid!=unset -k perm_mod
-a always,exit -F arch=b32 -S chown,fchown,fchownat,lchown -F auid>=1000 -F auid!=unset -k perm_mod
-a always,exit -F arch=b64 -S setxattr,lsetxattr,fsetxattr,removexattr,lremovexattr,fremovexattr -F auid>=1000 -F auid!=unset -k perm_mod
-a always,exit -F arch=b32 -S setxattr,lsetxattr,fsetxattr,removexattr,lremovexattr,fremovexattr -F auid>=1000 -F auid!=unset -k perm_mod

## =========================================================
## Unsuccessful File Access Attempts (EACCES/EPERM)
## =========================================================
-a always,exit -F arch=b64 -S creat,open,openat,truncate,ftruncate -F exit=-EACCES -F auid>=1000 -F auid!=unset -k access
-a always,exit -F arch=b32 -S creat,open,openat,truncate,ftruncate -F exit=-EACCES -F auid>=1000 -F auid!=unset -k access
-a always,exit -F arch=b64 -S creat,open,openat,truncate,ftruncate -F exit=-EPERM -F auid>=1000 -F auid!=unset -k access
-a always,exit -F arch=b32 -S creat,open,openat,truncate,ftruncate -F exit=-EPERM -F auid>=1000 -F auid!=unset -k access

## =========================================================
## Successful File System Mounts
## =========================================================
-a always,exit -F arch=b64 -S mount -F auid>=1000 -F auid!=unset -k mounts
-a always,exit -F arch=b32 -S mount -F auid>=1000 -F auid!=unset -k mounts

## =========================================================
## File Deletions and Renaming
## =========================================================
-a always,exit -F arch=b64 -S unlink,unlinkat,rename,renameat -F auid>=1000 -F auid!=unset -k delete
-a always,exit -F arch=b32 -S unlink,unlinkat,rename,renameat -F auid>=1000 -F auid!=unset -k delete

## =========================================================
## Kernel Module Loading and Unloading (finit_module added)
## =========================================================
-w /usr/sbin/insmod -p x -k modules
-w /usr/sbin/rmmod -p x -k modules
-w /usr/sbin/modprobe -p x -k modules
-a always,exit -F arch=b64 -S init_module,delete_module,finit_module -k modules
-a always,exit -F arch=b32 -S init_module,delete_module,finit_module -k modules

## =========================================================
## Make the Audit Configuration Immutable
## (Must be the last rule; prevents modification without reboot)
## =========================================================
-e 2
```

**Puis rechargez auditd et appliquez les règles**

```bash
sudo augenrules --load
sudo systemctl restart auditd || true
```

### **5.5 Configuration du Démon Auditd**

Afin de prévenir la perte de journaux et de configurer les avertissements liés à l'espace disque, éditez `/etc/audit/auditd.conf` :

```bash
sudo sed -i 's/^max_log_file_action.*/max_log_file_action = keep_logs/' /etc/audit/auditd.conf
sudo sed -i 's/^space_left_action.*/space_left_action = email/' /etc/audit/auditd.conf
sudo sed -i 's/^action_mail_acct.*/action_mail_acct = root/' /etc/audit/auditd.conf
sudo sed -i 's/^admin_space_left_action.*/admin_space_left_action = halt/' /etc/audit/auditd.conf
sudo systemctl restart auditd
```

### **5.6 Configuration de Journald**

Afin de conserver les logs localement et limiter l'empreinte disque via la compression.

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
sudo bash -c 'cat <<EOF > /etc/systemd/journald.conf.d/60-cis-journald.conf
[Journal]
Compress=yes
Storage=persistent
ForwardToSyslog=no
EOF'
sudo systemctl restart systemd-journald
```

## **Phase 6: Bannières et Avertissements Légaux**

La présence d'un avertissement légal (bannière) est indispensable avant toute authentification pour se prémunir juridiquement.

**Action :** Appliquez le message suivant aux fichiers /etc/motd, /etc/issue et /etc/issue.net.

```
*******************************************************************************
 AVERTISSEMENT: Accès autorisé uniquement.
 Ce système est surveillé à des fins de sécurité. Tout accès ou utilisation
 non autorisé(e) de ce système est interdit(e) et peut entraîner des sanctions
 pénales ou civiles.

 https://changeme.com
 *******************************************************************************
```

**Action :** Sécurisez les permissions de ces fichiers.

```bash
sudo chown root:root /etc/motd /etc/issue /etc/issue.net
sudo chmod 644 /etc/motd /etc/issue /etc/issue.net
```

## **Phase 7: Suppression des Services Inutiles***

Un serveur sécurisé ne doit faire tourner que ce dont il a besoin.

### **7.1 Suppression de l'interface graphique (GDM/X11)**

```bash
sudo dnf remove -y gdm; sudo dnf remove 'xserver-*' 'libx11-.*'
 sudo dnf autoremove -y
```

### **7.2 Nettoyage des services réseaux obsolètes ou non requis**

L'exécution des commandes de nettoyage supprimera les services listés ci-dessous, qui représentent un risque s'ils ne sont pas explicitement requis :

- **Partage de fichiers et protocoles hérités :**
- vsftpd, ftp, tnftp : Serveurs/clients FTP (les identifiants transitent en clair).
- tftpd-hpa : Serveur TFTP (Trivial FTP, ne possède aucune authentification).
- nfs-kernel-server : Serveur de partage réseau NFS (souvent mal sécurisé par défaut).
- samba, smbd : Partage de fichiers Windows (SMB/CIFS).
- autofs : Service de montage automatique de systèmes de fichiers distants.
- **Services de découverte et d'impression :**
- avahi-daemon : Service Zeroconf (mDNS/DNS-SD) utile pour les ordinateurs de bureau, mais dangereux sur un serveur.
- cups : Système d'impression (Common UNIX Printing System), inutile sur un serveur classique.
- bluez : Daemon Bluetooth, augmentant la surface d'attaque physique/sans fil.
- **Annuaire et authentification :**
- slapd, ldap-utils : Serveur d'annuaire LDAP et ses outils.
- ypserv : Serveur NIS (Network Information Service), protocole hérité obsolète et non chiffré.
- **Infrastructure réseau et messagerie :**
- bind9, named, dnsmasq : Serveurs et relais DNS.
- isc-dhcp-server : Serveur d'adresses IP DHCP.
- rpcbind : Mappeur de ports RPC, souvent ciblé pour des attaques DDoS par amplification.
- dovecot-imapd, dovecot-pop3d : Serveurs de réception d'emails.
- **Protocoles distants en clair :**
- rsh-client, talk, telnet, inetutils-telnet : Utilitaires de communication hérités sans chiffrement (à remplacer par SSH).
- **Serveurs Web / Proxy (si non requis par le rôle du serveur) :**
- apache2, nginx : Serveurs web.
- squid : Serveur proxy.
- **Divers :**
- snmp : Protocole de supervision réseau (les v1 et v2c transmettent en clair).
- rsync : Outil de synchronisation, s'il tourne en mode daemon autonome.
- xinetd : Super-serveur réseau étendu, de nos jours remplacé par systemd.Exécutez ce script pour arrêter et supprimer massivement ces services de manière tolérante aux erreurs (ignorera ceux qui ne sont pas installés) :

```bash
sudo dnf update -qq && \
 sudo dnf remove -y gdm tftp-server nfs-utils samba samba-client autofs \
 avahi cups bluez-libs bind bind-utils dnsmasq dhcp-server rpcbind dovecot \
 rsh talk telnet-server httpd nginx squid net-snmp rsync xinetd && \
 sudo dnf remove -y -qq && \
 sudo dnf clean all
```

### **7.3 Sécurisation du serveur Mail (MTA)**

Si le serveur ne sert pas de relais mail, le MTA (Postfix/Exim) ne doit écouter que sur localhost (127.0.0.1).

- **Vérification**: `sudo ss -plntu | grep -E ":(25|465|587)"`
- **Remédiation (Exemple Postfix) :** Dans `/etc/postfix/main.cf`, définissez inet_interfaces = loopback-only, puis redémarrez Postfix.

### **7.4 Corriger les permission de crontab**

```bash
sudo chown root:root /etc/{crontab,cron.hourly,cron.daily,cron.weekly,cron.monthly,cron.d}
sudo chmod og-rwx /etc/{crontab,cron.hourly,cron.daily,cron.weekly,cron.monthly,cron.d}
```

### **7.5 Restreindre l'accès à Cron et At**

Supprimer les fichiers `deny` et créer les fichiers `allow` restreints à l'utilisateur root.

```bash
sudo rm -f /etc/cron.deny /etc/at.deny
sudo bash -c 'echo "root" > /etc/cron.allow'
sudo bash -c 'echo "root" > /etc/at.allow'
sudo chmod 600 /etc/cron.allow /etc/at.allow
```

## **Phase 8: Politiques Cryptographiques Globales**

Contrairement aux systèmes où l'administrateur doit manuellement expurger les algorithmes obsolètes de chaque démon (SSH, Apache, Nginx, postfix), RHEL introduit l'outil update-crypto-policies. Ce mécanisme force de manière descendante (Top-Down) l'application d'un référentiel cryptographique standardisé à toutes les bibliothèques centrales de l'OS (OpenSSL, GnuTLS, NSS, libssh).

Trois niveaux de profils principaux structurent l'approche:

- **DEFAULT** : Niveau sécurisé actuel, interdisant TLS 1.0/1.1 et limitant sévèrement les algorithmes vieillissants.
- **FUTURE** : Posture avant-gardiste. Exige un hachage asymétrique de 3072 bits minimum et anticipe les algorithmes de chiffrement post-quantiques (Post-Quantum Cryptography). Il peut engendrer de profonds problèmes d'interopérabilité avec les systèmes tiers (Legacy).
- **FIPS** : Profil conformant aux normes strictes *Federal Information Processing Standard 140* du gouvernement américain. Il déclenche la vérification d'auto-intégrité (Self-Tests) à chaque appel de module cryptographique et injecte le paramètre fips=1 dans l'amorceur de boot. La bascule en mode FIPS sur un système déjà instancié peut paralyser l'architecture si les clés asymétriques existantes (IdM/Kerberos) n'ont pas été préalablement regénérées.

L'utilisation de politiques cryptographiques dépréciées (LEGACY), l'usage du hachage de signature algorithmique SHA-1 (qui, affaibli par les collisions, est compromis depuis plus d'une décennie), ainsi que l'utilisation du mode d'opération de bloc de chiffrement (Cipher Block Chaining - CBC) pour SSH, tristement célèbre pour ses vulnérabilités aux attaques de type *Padding Oracle sont* impérativement interdite.

La commande pour assainir globalement la cryptographie et désactiver les maillons faibles sans recourir au mode FIPS complexe est la suivante :

```bash
sudo update-crypto-policies --set DEFAULT:NO-SHA1:NO-SSHCBC
sudo reboot
```

## **Phase 9: Sécurisation du Réseau et Pare-feu**

### **9.1 Désactivation des protocoles réseaux exotiques**

```bash
sudo bash -c 'cat <<EOF > /etc/modprobe.d/hardening-networking.conf
 install dccp /bin/false
 blacklist dccp
 install tipc /bin/false
 blacklist tipc
 install rds /bin/false
 blacklist rds
 install sctp /bin/false
 blacklist sctp
 EOF'
 sudo modprobe -r dccp tipc rds sctp 2>/dev/null
```

### **9.2 Durcissement TCP/IP via Sysctl**

Prévenir le spoofing IP, ignorer les requêtes ICMP broadcast, et activer les SYN cookies.

```bash
sudo bash -c 'cat <<EOF >> /etc/sysctl.d/60-network-hardening.conf
 net.ipv4.ip_forward = 0
 net.ipv6.conf.all.forwarding = 0
 net.ipv6.conf.default.forwarding = 0
 net.ipv4.conf.all.accept_redirects = 0
 net.ipv4.conf.default.accept_redirects = 0
 net.ipv4.conf.all.secure_redirects = 0
 net.ipv4.conf.default.secure_redirects = 0
 net.ipv4.conf.all.send_redirects = 0
 net.ipv4.conf.default.send_redirects = 0
 net.ipv6.conf.all.accept_redirects = 0
 net.ipv6.conf.default.accept_redirects = 0
 net.ipv4.conf.all.rp_filter = 1
 net.ipv4.conf.default.rp_filter = 1
 net.ipv4.conf.all.log_martians = 1
 net.ipv4.conf.default.log_martians = 1
 net.ipv4.tcp_syncookies = 1
 net.ipv4.icmp_echo_ignore_broadcasts = 1
 net.ipv4.icmp_ignore_bogus_error_responses = 1
 net.ipv6.conf.all.accept_ra = 0
 net.ipv6.conf.default.accept_ra = 0
 net.ipv4.conf.all.accept_source_route = 0
 net.ipv4.conf.default.accept_source_route = 0
 net.ipv6.conf.all.accept_source_route = 0
 net.ipv6.conf.default.accept_source_route = 0
 net.ipv4.conf.default.log_martians = 1
 net.ipv4.conf.all.log_martians = 1
 EOF'
 sudo sysctl --system
```

### **9.4 Configuration du Pare-feu (Firewalld / nftables)**

Application d’une politique de type "Deny All / Allow Exception".

```bash
# 1. Set default policies to DROP for incoming, outgoing, and routed traffic
sudo firewall-cmd --permanent --zone=public --set-target=DROP
sudo firewall-cmd --permanent --policy=default-policy-outbound --set-target=DROP
sudo firewall-cmd --permanent --policy=default-policy-routed --set-target=DROP

# 2. Deny external traffic pretending to be from loopback (Anti-spoofing)
sudo firewall-cmd --permanent --zone=drop --add-source=127.0.0.0/8
sudo firewall-cmd --permanent --zone=drop --add-source=::1

# 3. Allow incoming SSH (TCP 22) from anywhere
sudo firewall-cmd --permanent --zone=public --add-port=22/tcp

# 4. Allow outgoing HTTP (80/tcp), HTTPS (443/tcp), DNS (53/tcp & 53/udp), and NTP (123/udp)
# We use rich rules to explicitly permit outbound traffic since the global outbound policy is DROP
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" destination port port="80" protocol="tcp" accept'
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" destination port port="443" protocol="tcp" accept'
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" destination port port="53" protocol="tcp" accept'
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" destination port port="53" protocol="udp" accept'
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" destination port port="123" protocol="udp" accept'

# 5. Reload firewalld to apply and enforce all changes (Equivalent to ufw enable)
sudo systemctl enable --now firewalld
sudo firewall-cmd --reload
```

## **Phase 10: Authentification, Contrôle d'Accès et SSH**

### **10.1 Sécurisation du démon SSH**

Création d'un profil sécurisé désactivant l'accès root et forçant l'usage de clés.

```bash
sudo bash -c 'cat <<EOF > /etc/ssh/sshd_config.d/06-ssh-hardening.conf
 Banner /etc/issue.net
 PermitRootLogin no
 PasswordAuthentication yes
 PermitEmptyPasswords no
 MaxAuthTries 4
 ClientAliveInterval 30
 ClientAliveCountMax 3
 LoginGraceTime 60
 MaxAuthTries 4
 X11Forwarding no
 AllowTcpForwarding no
 IgnoreRhosts yes
 HostbasedAuthentication no
 UsePAM yes
 PermitUserEnvironment no
 DisableForwarding yes
 MaxStartups 10:30:60
 LogLevel VERBOSE
 Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
 MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
 KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group14-sha256,diffie-hellman-group16-sha512
 EOF'
 sudo systemctl restart ssh
```

Correction des permissions

```bash
sudo chmod u-x,og-rwx /etc/ssh/sshd_config && \
 sudo chown root:root /etc/ssh/sshd_config && \
 sudo find /etc/ssh/sshd_config.d -type f -exec chmod u-x,og-rwx {} + 2>/dev/null && \
 sudo find /etc/ssh/sshd_config.d -type f -exec chown root:root {} + 2>/dev/null
```

### **10.2 Durcissement de Sudo**

Activer les logs Sudo pour l'audit.

```bash
sudo bash -c 'cat <<EOF > /etc/sudoers.d/00-cis-hardening
 Defaults use_pty
 Defaults logfile="/var/log/sudo.log"
 EOF'
```

### **10.3 Politique des Mots de Passe (PAM)**

La gestion d'identité PAM (Pluggable Authentication Modules) sous Red Hat est pilotée par l'utilitaire authselect. Toute modification manuelle des fichiers /etc/pam.d/system-auth ou /etc/pam.d/password-auth est écrasée lors des mises à jour ; par conséquent, l'usage de commandes authselect est obligatoire.

Le durcissement de l'authentification couvre plusieurs aspects impératifs 1 :

- **Verrouillage lors des attaques par force brute (pam_faillock)** : Bloque automatiquement un compte utilisateur après un nombre défini de tentatives infructueuses (généralement entre 3 et 5 tentatives), durant une période de temporisation (par exemple 15 minutes). Le compte root doit impérativement être soumis à cette règle.
- **Complexité des mots de passe (pam_pwquality)** : Impose des classes de caractères strictes, interdit la présence du nom d'utilisateur dans le mot de passe, vérifie les condensats à l'aide de dictionnaires via cracklib, et interdit des séquences consécutives triviales (ex. 1234, abcd).
- **Renouvellement et historique (pam_pwhistory)** : Consigne les hachages SHA-512 ou yescrypt des anciens mots de passe pour empêcher un utilisateur de réutiliser l'une de ses 5 précédentes combinaisons, forçant ainsi une véritable rotation.1
- **Répudiation des mots de passe vides (without-nullok)** : L'option nullok autorise des connexions dépourvues de mot de passe. Le CIS interdit ce paramètre de manière transversale

```bash
sudo authselect select sssd with-faillock with-pwhistory --force
```

Configurer le verrouillage par force brute (`pam_faillock`)

```bash
sudo mkdir -p /etc/security/faillock.conf.d
sudo bash -c 'cat << EOF > /etc/security/faillock.conf.d/60-hardening-faillock.conf
# Configuration du verrouillage des comptes (pam_faillock)
deny = 3
unlock_time = 900
even_deny_root
EOF'
```

**Configurer la complexité (pwquality)**

```bash
sudo mkdir -p /etc/security/pwquality.conf.d
sudo bash -c 'cat <<EOF > /etc/security/pwquality.conf.d/60-hardening-pw.conf
difok = 2
minlen = 14
minclass = 3
dcredit = -1
ucredit = -1
ocredit = -1
lcredit = -1
maxrepeat = 3
maxsequence = 3
dictcheck = 1
enforce_for_root = yes
EOF'
```

**Configurer l'historique des mots de passe (pwhistory)**

```bash
sudo bash -c 'cat <<EOF > /etc/security/pwhistory.conf
# Configuration de l'historique (pam_pwhistory)
remember = 24
enforce_for_root = yes
EOF'
```

Verification

```bash
sudo authselect check
```

### **10.3.1 Sécurisation des Fichiers d'Authentification**

Restreindre les permissions sur les fichiers sensibles du système :

```bash
sudo chown root:shadow /etc/shadow /etc/shadow- /etc/gshadow /etc/gshadow-
sudo chmod 0640 /etc/shadow /etc/shadow- /etc/gshadow /etc/gshadow-

sudo chown root:root /etc/shells
sudo chmod 0644 /etc/shells

sudo touch /etc/security/opasswd
sudo chown root:root /etc/security/opasswd
sudo chmod 0600 /etc/security/opasswd
```

**Actions complémentaires :**

Éditez /etc/login.defs pour configurer l'expiration :

```jsx
PASS_MAX_DAYS 90
PASS_MIN_DAYS 1
PASS_WARN_AGE 7
ENCRYPT_METHOD SHA512
```

### **10.4 Vérification d'intégrité des comptes**

- Vérifier que seul root possède l'UID 0 :

```bash
awk -F: '$3 == 0 {print $1}' /etc/passwd
```

- S'assurer que les comptes systèmes n'ont pas de shell valide: Le resultat devrait contenir “root” et possiblement “sync” qui a pour shell /bin/sync

```bash
awk -F: '$3 < 1000 && $7 != "/sbin/nologin" && $7 != "/bin/false" {print $1}' /etc/passwd
```

### **10.5 Sécurité des profils utilisateurs (TMOUT et umask)**

Définir une expiration d'inactivité (TMOUT) et un masque de création de fichiers restrictif.

```bash
# 1. Configuration pour les shells standards (sh, bash, ksh)
sudo bash -c 'cat <<EOF > /etc/profile.d/99-hardening.sh
readonly TMOUT=900
export TMOUT
umask 027
EOF'

# 2. Configuration pour les shells alternatifs (csh, tcsh)
sudo bash -c 'cat <<EOF > /etc/profile.d/99-hardening.csh
set autologout=15
set -r autologout
umask 027
EOF'

# 3. Application des bons droits (lecture uniquement)
sudo chmod 644 /etc/profile.d/99-hardening.sh
sudo chmod 644 /etc/profile.d/99-hardening.csh
```

## **Phase 11: Appliquer Redemarrage final**

```bash
sudo reboot
```
