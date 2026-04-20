# Chapitres Du Rapport De Stage

## 1. Structure Du Projet

Le projet SEC-LAb est organise autour d'une architecture Ansible modulaire destinee a automatiser l'audit, le durcissement et la verification de conformite de plusieurs types de cibles : serveurs Linux, postes Windows et terminaux Android via Termux.

La structure generale du projet est la suivante :

```text
SEC-LAb/
├── ansible.cfg
├── inventory.ini
├── requirements.yml
├── group_vars/
│   ├── all/
│   │   ├── vars.yml
│   │   └── vault.yml
│   ├── linux_servers/
│   │   ├── vars.yml
│   │   └── vault.yml
│   ├── phones/
│   │   ├── vars.yml
│   │   └── vault.yml
│   └── windows_machines.yml
├── playbooks/
│   ├── baseline_scan.yml
│   ├── full_pipeline.yml
│   ├── generate_report.yml
│   ├── harden.yml
│   ├── linux_check.yml
│   ├── linux_harden.yml
│   ├── linux_remediate.yml
│   ├── linux_scan.yml
│   ├── remediate.yml
│   ├── rescan.yml
│   ├── site.yml
│   ├── windows_check.yml
│   ├── windows_harden.yml
│   ├── windows_remediate.yml
│   └── windows_scan.yml
├── roles/
│   ├── common/
│   ├── linux_baseline/
│   ├── linux_hardening/
│   ├── openscap_scan/
│   ├── openscap_remediate/
│   ├── openscap_rescan/
│   ├── windows_baseline/
│   ├── windows_hardening/
│   ├── hardeningkitty_scan/
│   ├── hardeningkitty_remediate/
│   ├── hardeningkitty_rescan/
│   └── phone_check/
├── reports/
│   ├── linux/
│   └── windows/
└── docs/
    ├── PROJECT.md
    └── CHAPITRES_RAPPORT_STAGE.md
```

Cette organisation repose sur plusieurs principes. Le fichier `inventory.ini` centralise les machines a administrer. Le repertoire `group_vars/` contient les variables par groupe de machines ainsi que les secrets chiffres avec Ansible Vault. Les playbooks du repertoire `playbooks/` decrivent les grandes etapes du pipeline de securisation. Les roles du repertoire `roles/` encapsulent la logique metier par domaine technique, par exemple le durcissement Linux, le scan OpenSCAP ou l'audit Windows avec HardeningKitty. Enfin, les rapports HTML et XML produits pendant les scans sont centralises dans le repertoire `reports/`.

Le pipeline mis en oeuvre par le projet suit la logique suivante : verification de connectivite, scan initial, application des correctifs de durcissement, remediation automatique complementaire, puis rescan final pour mesurer le niveau de conformite obtenu vis-a-vis des referentiels CIS et ANSSI.

## 2. Installation D'Ansible

### 2.1 Objectif

Ansible est l'outil d'automatisation retenu pour centraliser l'execution des controles de securite, le durcissement des systemes et la collecte des rapports. Dans le cadre de ce projet, Ansible joue le role de noeud de controle et orchestre les actions vers les machines Linux, Windows et Android via SSH.

### 2.2 Prerequis

Avant l'installation, les prerequis suivants doivent etre verifies sur la machine de controle :

- Un systeme Linux de type Ubuntu 22.04 ou equivalent.
- Un acces Internet pour telecharger les dependances.
- Un compte utilisateur ayant les droits sudo.
- Python 3 installe sur la machine de controle.
- Une connectivite reseau vers les hotes cibles.

### 2.3 Procedure D'Installation

L'installation d'Ansible sur Ubuntu peut etre effectuee avec les commandes suivantes :

```bash
sudo apt update
sudo apt install -y software-properties-common
sudo apt install -y python3 python3-pip sshpass git
sudo apt install -y ansible
ansible --version
```

Dans le cas ou une version plus recente d'Ansible est necessaire, l'installation via `pip` reste possible :

```bash
python3 -m pip install --user --upgrade pip
python3 -m pip install --user ansible
~/.local/bin/ansible --version
```

### 2.4 Installation Des Collections Utiles

Le projet repose egalement sur certaines collections Ansible externes, notamment pour la gestion de Windows et de certaines fonctions systeme. Une fois place a la racine du projet, l'installation se fait ainsi :

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2.5 Configuration Du Projet

Le fichier `ansible.cfg` permet de definir les parametres globaux du projet, notamment le chemin de l'inventaire, le dossier des roles, l'utilisation d'un fichier de mot de passe Vault, ou encore l'optimisation des connexions SSH. Une fois Ansible installe, il convient de verifier les elements suivants :

- La presence du fichier `inventory.ini`.
- La coherence des variables dans `group_vars/`.
- La presence du fichier `.vault_pass` sur la machine locale uniquement.
- Les droits d'acces aux hotes cibles via SSH.

### 2.6 Verification Du Bon Fonctionnement

Une premiere verification fonctionnelle peut etre realisee a l'aide des playbooks de connectivite :

```bash
ansible-playbook playbooks/linux_check.yml
ansible-playbook playbooks/windows_check.yml
ansible-playbook playbooks/phone_check.yml
```

Cette etape permet de confirmer que le noeud Ansible peut joindre correctement les differents equipements avant de lancer les scans ou le durcissement.

## 3. Installation De Wazuh En Mode Single-Node

### 3.1 Objectif

Wazuh est une plateforme open source de supervision et de securite permettant la collecte de logs, la detection d'intrusion, le suivi d'integrite, la correlation d'evenements et la supervision des agents deployes sur les machines. Dans ce projet, l'architecture retenue est un mode single-node, c'est-a-dire une installation tout-en-un integree sur une seule machine. Ce mode est adapte a un laboratoire, a un POC ou a un environnement de stage.

### 3.2 Composants Installes

Une installation Wazuh single-node regroupe generalement les composants suivants sur la meme machine :

- Le serveur Wazuh.
- L'indexeur Wazuh.
- Le dashboard Wazuh.
- Les services necessaires a la gestion des agents et des alertes.

### 3.3 Prerequis Techniques

Les prerequis recommandes sont les suivants :

- Une machine Ubuntu 22.04 64 bits.
- Au moins 4 vCPU.
- Au moins 8 Go de RAM pour un laboratoire confortable.
- Au moins 50 Go d'espace disque.
- Un acces Internet.
- Un nom d'hote correctement defini.
- Les ports reseau ouverts selon l'architecture retenue.

### 3.4 Installation

La methode la plus simple pour un deploiement single-node consiste a utiliser le script officiel Wazuh. Les commandes suivantes illustrent le processus :

```bash
sudo apt update
sudo apt install -y curl tar
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.7/config.yml
sudo bash wazuh-install.sh --generate-config-files
sudo bash wazuh-install.sh --wazuh-indexer node-1
sudo bash wazuh-install.sh --wazuh-server wazuh-1
sudo bash wazuh-install.sh --wazuh-dashboard dashboard
```

Dans un cadre de rapport, il est pertinent de preciser que cette sequence peut varier legerement selon la version exacte de Wazuh et la methode officielle en vigueur au moment du deploiement.

### 3.5 Verification Des Services

Apres l'installation, il faut verifier que les services Wazuh demarrent correctement :

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

L'acces au tableau de bord se fait ensuite depuis un navigateur via l'adresse du serveur, par exemple :

```text
https://IP_DU_SERVEUR_WAZUH
```

### 3.6 Interet De Wazuh Dans Le Projet

Dans le contexte du projet SEC-LAb, Wazuh peut jouer un role complementaire aux scans OpenSCAP et HardeningKitty. Il permet notamment de centraliser les evenements de securite, de superviser les machines durcies, de remonter les alertes et de maintenir une vision continue de l'etat du parc au-dela d'un simple audit ponctuel.

## 4. Installation Des Agents Wazuh

### 4.1 Objectif

Les agents Wazuh permettent de remonter vers le serveur central les logs systeme, les evenements de securite, l'etat d'integrite des fichiers et diverses informations de supervision. Le deploiement des agents constitue donc une etape essentielle pour assurer une visibilite continue sur les machines surveillees.

### 4.2 Installation D'Un Agent Wazuh Sous Linux

Sur une machine Ubuntu, l'installation peut se faire avec le depot officiel Wazuh. Exemple de procedure :

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo WAZUH_MANAGER="IP_DU_SERVEUR_WAZUH" apt install -y wazuh-agent
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Apres installation, le service peut etre verifie avec :

```bash
sudo systemctl status wazuh-agent
```

### 4.3 Installation D'Un Agent Wazuh Sous Windows

Sous Windows, l'installation peut etre effectuee a partir du package MSI fourni par Wazuh. Dans un cadre de deploiement manuel, la commande PowerShell suivante peut etre utilisee :

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi -OutFile wazuh-agent.msi
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="IP_DU_SERVEUR_WAZUH"
Start-Service Wazuh
Set-Service Wazuh -StartupType Automatic
```

La verification du service se fait ensuite via PowerShell :

```powershell
Get-Service Wazuh
```

### 4.4 Enregistrement Et Validation Cote Serveur

Une fois les agents installes, leur etat peut etre valide depuis le serveur Wazuh ou depuis le dashboard. L'objectif est de verifier que chaque hote apparait bien comme connecte, remonte ses informations et commence a transmettre ses journaux.

### 4.5 Interet Des Agents Dans Le Cadre Du Stage

Pour un rapport de stage, il est pertinent d'expliquer que les agents Wazuh prolongent la logique d'automatisation du projet. Alors qu'Ansible applique des configurations et que les scanners mesurent la conformite, Wazuh fournit une supervision continue apres le deploiement des correctifs. Il devient ainsi possible de verifier qu'un systeme durci reste conforme dans le temps et qu'aucune derive de configuration ne survient.

## 5. Installation D'OpenSCAP

### 5.1 Objectif

OpenSCAP est l'outil retenu pour l'audit de conformite des systemes Linux. Il s'appuie sur des contenus SCAP standards, notamment le SCAP Security Guide, pour evaluer les machines selon des profils de securite tels que CIS ou ANSSI. Dans ce projet, OpenSCAP est utilise a trois niveaux : scan initial, remediation automatique et rescan de verification.

### 5.2 Paquets A Installer

Sur Ubuntu 22.04, les paquets utilises dans le projet sont les suivants :

```bash
sudo apt update
sudo apt install -y libopenscap8 ssg-debderived bzip2
```

Selon les distributions, le binaire principal est `oscap`. Le paquet `ssg-debderived` fournit les profils et datastreams preconfigures pour les distributions Debian et Ubuntu.

### 5.3 Verification De L'Installation

Les commandes ci-dessous permettent de verifier que l'outil et les contenus sont presents :

```bash
oscap version
ls /usr/share/xml/scap/ssg/content/
oscap info /usr/share/xml/scap/ssg/content/ssg-ubuntu2204-ds.xml
```

### 5.4 Exemple De Scan De Conformite

Un exemple de scan CIS peut etre effectue avec la commande suivante :

```bash
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_level1_server \
  --report cis_report.html \
  --results cis_results.xml \
  /usr/share/xml/scap/ssg/content/ssg-ubuntu2204-ds.xml
```

De la meme maniere, un scan selon le profil ANSSI peut etre lance en adaptant le profil utilise :

```bash
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_anssi_bp28_enhanced \
  --report anssi_report.html \
  --results anssi_results.xml \
  /usr/share/xml/scap/ssg/content/ssg-ubuntu2204-ds.xml
```

### 5.5 Remediation Automatique

L'un des avantages d'OpenSCAP est la possibilite d'appliquer automatiquement certaines corrections. Dans le projet, cette possibilite est exploitee via l'option `--remediate` :

```bash
oscap xccdf eval \
  --remediate \
  --profile xccdf_org.ssgproject.content_profile_cis_level1_server \
  --report cis_remediation.html \
  --results cis_remediation.xml \
  /usr/share/xml/scap/ssg/content/ssg-ubuntu2204-ds.xml
```

Cette approche ne remplace pas totalement un role Ansible de durcissement, mais elle permet de completer le traitement en appliquant automatiquement des correctifs supplementaires fournis par les contenus SCAP.

### 5.6 Integration Dans Le Projet SEC-LAb

Dans le projet developpe, OpenSCAP est integre au travers de trois roles :

- `openscap_scan` pour le scan initial.
- `openscap_remediate` pour la remediation automatique.
- `openscap_rescan` pour la verification finale.

Cette integration permet d'inscrire l'audit de conformite dans un pipeline complet : mesurer, corriger, reevaluer, puis produire des rapports HTML et XML exploitables dans le cadre du projet et du rapport de stage.

## 6. Description Detaillee Des Playbooks

Cette section decrit le role de chaque playbook present dans le projet afin de clarifier la logique d'orchestration.

### 6.1 Playbooks Principaux

- `playbooks/full_pipeline.yml` : pipeline complet de bout en bout. Il enchaine la connectivite, le scan baseline, le durcissement manuel, la remediation automatique, puis le rescan final.
- `playbooks/baseline_scan.yml` : execute les audits initiaux avant correction. Linux est scanne avec OpenSCAP (CIS + ANSSI) et Windows avec HardeningKitty (CIS).
- `playbooks/harden.yml` : applique le durcissement sur Linux et Windows. Il combine les roles de hardening manuel puis les roles de remediation automatique.
- `playbooks/remediate.yml` : lance uniquement la phase de remediation automatique (OpenSCAP `--remediate` pour Linux et HardeningKitty HailMary pour Windows).
- `playbooks/rescan.yml` : relance les scans apres durcissement/remediation pour mesurer l'amelioration de conformite.
- `playbooks/generate_report.yml` : consolide les resultats baseline et post-hardening et produit un rapport de synthese HTML.
- `playbooks/site.yml` : playbook global de type "site" pour executer les controles de base sur tous les groupes selon une logique centralisee.

### 6.2 Playbooks Linux

- `playbooks/linux_check.yml` : test de connectivite et verification de base des serveurs Linux.
- `playbooks/linux_scan.yml` : scan baseline Linux avec OpenSCAP.
- `playbooks/linux_harden.yml` : durcissement Linux (manuel + remediation).
- `playbooks/linux_remediate.yml` : remediation Linux uniquement via OpenSCAP.

### 6.3 Playbooks Windows

- `playbooks/windows_check.yml` : test de connectivite et controle baseline des machines Windows via SSH/PowerShell.
- `playbooks/windows_scan.yml` : scan baseline Windows via HardeningKitty.
- `playbooks/windows_harden.yml` : durcissement Windows (manuel + HailMary).
- `playbooks/windows_remediate.yml` : remediation Windows uniquement via HardeningKitty HailMary.

### 6.4 Playbook Android

- `playbooks/phone_check.yml` : test de connectivite et collecte d'informations minimales sur les telephones Android (Termux SSH).

## 7. Description Detaillee Des Roles

Cette section decrit chaque role Ansible et sa fonction dans la chaine de securisation.

### 7.1 Roles Communs Et Baseline

- `roles/common` : verifie la connectivite SSH sur toutes les cibles avec un fallback si le module `ping` n'est pas disponible.
- `roles/linux_baseline` : collecte des informations systeme Linux avant durcissement (OS, kernel, services, ports, packages).
- `roles/windows_baseline` : collecte des informations de base Windows avant durcissement (version OS, services, ports, firewall).
- `roles/phone_check` : verifie la disponibilite des terminaux Android/Termux et remonte des informations de base.

### 7.2 Roles Linux (OpenSCAP Et Hardening)

- `roles/openscap_scan` : installe OpenSCAP/SSG, execute les scans baseline CIS et ANSSI, extrait les scores et recupere les rapports HTML/XML.
- `roles/openscap_remediate` : applique la remediation automatique OpenSCAP (`--remediate`) sur profils CIS et ANSSI, puis collecte les resultats.
- `roles/openscap_rescan` : execute les scans post-hardening pour valider le niveau de conformite final.
- `roles/linux_hardening` : role principal de durcissement Linux, decoupe en sous-fichiers thematiques.

Sous-modules de `roles/linux_hardening/tasks/` :

- `filesystem.yml` : options de montage securisees, desactivation de systemes de fichiers/protocoles non necessaires, reduction de la surface d'attaque.
- `sysctl.yml` : renforcement noyau et pile reseau (anti-spoofing, anti-redirect, restrictions kernel).
- `ssh.yml` : durcissement SSH (authentification, ciphers, restrictions d'acces, bannieres).
- `auth.yml` : politiques de mots de passe, verrouillage de compte, regles PAM.
- `services.yml` : arret/desactivation des services inutiles et reduction des composants exposes.
- `network.yml` : configuration pare-feu et parametres reseau de securite.
- `logging.yml` : auditd, rsyslog, retention et tracabilite des evenements.
- `permissions.yml` : durcissement des permissions sur fichiers critiques.
- `banners.yml` : bannieres legales et messages d'avertissement conformes.

### 7.3 Roles Windows (HardeningKitty Et Durcissement)

- `roles/hardeningkitty_scan` : deploie HardeningKitty, execute l'audit baseline CIS et genere les rapports CSV/HTML.
- `roles/hardeningkitty_remediate` : applique les correctifs automatiques via mode HailMary et recupere les journaux de remediation.
- `roles/hardeningkitty_rescan` : execute l'audit post-hardening pour verifier l'amelioration du score de conformite.
- `roles/windows_hardening` : role principal de durcissement Windows, compose de plusieurs sous-fichiers.

Sous-modules de `roles/windows_hardening/tasks/` :

- `account_policy.yml` : politiques de comptes et mots de passe (complexite, lockout, age, comptes par defaut).
- `audit_policy.yml` : activation des politiques d'audit avancees et des journaux de securite.
- `registry.yml` : configuration de cles registre de securite (UAC, SMB, LLMNR, protections systeme).
- `services.yml` : desactivation de services Windows non necessaires.
- `firewall.yml` : activation/renforcement du firewall Windows sur tous les profils.
- `network.yml` : durcissement protocolaire reseau (SMBv1, WPAD, NetBIOS, DNS settings).
- `user_rights.yml` : attribution des droits utilisateurs sensibles et restrictions privilegiees.

## 8. Conclusion Technique

Le projet SEC-LAb met en oeuvre une chaine complete de securisation automatisee. Ansible assure l'orchestration des taches de configuration. OpenSCAP et HardeningKitty mesurent la conformite des systemes Linux et Windows selon des referentiels reconnus. Les phases de durcissement et de remediation permettent d'augmenter le niveau de securite de maniere reproductible. Enfin, Wazuh ajoute une dimension de supervision continue, utile pour prolonger les benefices du durcissement dans le temps.

Dans un contexte de stage, cette architecture presente un interet pedagogique et technique important. Elle permet de montrer la complementarite entre automatisation, audit de conformite, remediation et supervision centralisee, tout en s'inscrivant dans une logique proche des attentes d'un environnement professionnel.