# Déploiement d'un serveur VoIP Asterisk 22 sur Debian 13

## Présentation du projet

Ce projet consiste à déployer et configurer un serveur de téléphonie IP (VoIP) basé sur **Asterisk 22**, compilé depuis ses sources sur une machine virtuelle Debian 13.

L'objectif est de mettre en place une infrastructure permettant l'enregistrement de clients SIP, la gestion des extensions téléphoniques et la réalisation d'appels entre plusieurs terminaux.

Le serveur utilise le module **PJSIP** pour gérer les communications SIP et a été testé avec le logiciel Zoiper sur Windows et Android.

## Objectifs techniques

- Installer et préparer un serveur Debian 13 minimal.
- Configurer une adresse IPv4 statique et l'accès SSH.
- Télécharger et compiler Asterisk depuis les sources.
- Installer les dépendances nécessaires à la compilation.
- Résoudre des problèmes de compatibilité de DAHDI avec les noyaux Linux récents.
- Installer et configurer Asterisk 22.
- Configurer les comptes SIP avec PJSIP.
- Définir un plan de numérotation avec `extensions.conf`.
- Tester l'enregistrement des clients et les communications VoIP.

## Architecture technique

### Environnement

| Élément | Configuration |
|---|---|
| Hyperviseur | VirtualBox |
| Système serveur | Debian 13 |
| Serveur VoIP | Asterisk 22.12.0 |
| Protocole de signalisation | SIP |
| Module SIP | PJSIP |
| Transport | UDP |
| Port SIP | 5060 |
| Interface réseau | Accès par pont |
| Adresse IP serveur | 192.168.1.95 |
| Client Windows | Zoiper |
| Client mobile | Zoiper Android |

### Dépendances compilées

| Composant | Version |
|---|---|
| DAHDI Linux Complete | 3.4.0+3.4.0 |
| libpri | 1.6.1 |
| Asterisk | 22.12.0 |

## Architecture logique

```text
                 Réseau local
                  192.168.1.0/24
                         |
                 +----------------+
                 |  Debian 13     |
                 |  Asterisk 22   |
                 |  192.168.1.95  |
                 |  UDP 5060      |
                 +----------------+
                         |
                 +-------+-------+
                 |               |
          +-------------+  +-------------+
          | PC Windows  |  | Smartphone  |
          | Zoiper      |  | Zoiper      |
          | Extension   |  | Extension   |
          | 123         |  | 124         |
          +-------------+  +-------------+
```

## 1. Préparation du serveur Debian

Création d'une machine virtuelle Debian 13 sans environnement graphique.

Configuration retenue :
- 2 vCPU
- 4 Go de RAM
- 20 Go de stockage
- Carte réseau en mode accès par pont

Préparation du système :
- Installation de sudo.
- Ajout de l'utilisateur au groupe sudo.
- Configuration de l'adresse IP.
- Installation du serveur OpenSSH.
- Vérification de la connectivité réseau depuis la machine hôte.

## 2. Préparation de la compilation

Création du répertoire de travail :

```bash
mkdir ~/asterisk-src
cd ~/asterisk-src
```

Téléchargement des sources :
- Asterisk 22
- DAHDI Linux Complete
- libpri

Extraction des archives et installation des outils de compilation :

```bash
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r)
```

Installation des dépendances nécessaires à DAHDI :

```bash
sudo apt install automake autoconf bison flex libtool \
libncurses5-dev patch sql
```

Installation des bibliothèques complémentaires :

```bash
sudo apt install libusb-1.0-0-dev libglib2.0-dev
```

## 3. Compilation et installation des dépendances

### DAHDI

La compilation initiale a rencontré des erreurs de compatibilité avec les noyaux Linux récents.

L'analyse des erreurs a permis d'identifier des incompatibilités dans trois fichiers sources.

Fichiers concernés :

- `dahdi-sysfs.c`
- `dahdi-sysfs-chan.c`
- `xbus-sysfs.c`

Une correction ciblée a été appliquée à quatre déclarations de structures en ajoutant le qualificatif `const` aux paramètres concernés.

Après correction :

```bash
make all
sudo make install
sudo make install-config
```

### libpri

Compilation et installation :

```bash
make
sudo make install
```

### Asterisk

Installation des dépendances manquantes identifiées lors de la configuration :

```bash
sudo apt install libedit-dev
sudo apt install libxml2-dev
sudo apt install libssl-dev
```

Configuration de la compilation :

```bash
./configure --with-jansson-bundled
```

Sélection des modules et codecs :

```bash
make menuselect
```

Activation des codecs audio français WAV, ULAW et ALAW, ainsi que leurs équivalents dans les ressources supplémentaires.

Compilation et installation :

```bash
make
sudo make install
```

Génération des fichiers de configuration et installation des exemples :

```bash
sudo make samples
sudo make config
```

Installation facultative de la rotation des journaux :

```bash
sudo make install-logrotate
```

## 4. Configuration d'Asterisk

Les deux principaux fichiers de configuration utilisés sont :

- `/etc/asterisk/pjsip.conf`
- `/etc/asterisk/extensions.conf`

Avant toute modification, une sauvegarde des fichiers d'origine a été réalisée.

### Configuration PJSIP

Le fichier `pjsip.conf` définit :
- Le transport SIP en UDP.
- Le port d'écoute 5060.
- Les endpoints.
- Les mécanismes d'authentification.
- Les comptes téléphoniques.
- Les paramètres audio.

Deux extensions ont été configurées :
- 123
- 124

Les configurations d'authentification sont associées à chaque endpoint.

### Plan de numérotation

Le fichier `extensions.conf` définit les règles de traitement des appels.

Un premier scénario de test a été mis en place :

```ini
[lab]
exten => 123,1,Answer()
exten => 123,2,Wait(2)
exten => 123,3,Playback(hello-world)
exten => 123,4,Hangup()
```

Ce scénario permet de vérifier le fonctionnement du serveur en déclenchant la lecture d'un message audio lors de l'appel de l'extension 123.

Une seconde configuration a ensuite été mise en place pour permettre les appels entre les deux extensions :

```ini
[lab]
exten => 123,1,Dial(PJSIP/${EXTEN},30)
exten => 123,2,Hangup()

exten => 124,1,Dial(PJSIP/${EXTEN},30)
exten => 124,2,Hangup()
```

## 5. Démarrage et vérifications

Démarrage du service :

```bash
sudo systemctl start asterisk
```

Vérification de son état :

```bash
sudo systemctl status asterisk
```

Installation des outils réseau :

```bash
sudo apt install net-tools
```

Vérification des ports en écoute :

```bash
sudo netstat -anup
```

Le port UDP 5060 est visible dans les résultats, associé au processus Asterisk.

Accès à la console Asterisk :

```bash
sudo /usr/sbin/asterisk -rvvvvv
```

Activation du suivi des requêtes PJSIP :

```text
pjsip set logger on
```

Ces commandes permettent d'observer les échanges SIP et de diagnostiquer les connexions des clients.

## 6. Tests et validation

### Test 1 : client Windows

Installation et configuration de Zoiper sur Windows.

Résultats observés :
- Détection du protocole SIP UDP.
- Enregistrement du compte 123.
- Appel de l'extension 123.
- Lecture correcte du message audio de test « Hello world ».

### Test 2 : deuxième extension

Création et configuration de l'extension 124 dans PJSIP.

Modification du plan de numérotation pour permettre les appels entre les deux comptes.

### Test 3 : client Android

Installation et configuration de Zoiper sur smartphone avec le compte 124.

Résultat observé :
- Détection du transport SIP UDP.
- Connexion du compte mobile.
- Réalisation d'un appel avec le serveur Asterisk.

## Résultats obtenus

Le serveur Asterisk 22.12.0 a été compilé et installé sur Debian 13.

Les configurations PJSIP et le plan de numérotation ont été mis en place. Les tests réalisés avec Zoiper sur Windows et Android ont permis de valider le fonctionnement de l'infrastructure de téléphonie IP.

## Compétences mobilisées

- Administration Linux Debian.
- Gestion des utilisateurs et des privilèges sudo.
- Configuration réseau IPv4.
- Administration SSH.
- Compilation depuis les sources.
- Gestion des dépendances et bibliothèques Linux.
- Diagnostic et résolution d'erreurs de compilation.
- Administration de services Linux avec systemd.
- Configuration d'un serveur VoIP.
- Protocoles SIP et transport UDP.
- Configuration PJSIP et gestion des extensions.
- Analyse des échanges réseau et journaux applicatifs.

## Sécurité et précautions

Ce projet est réalisé dans un environnement de laboratoire.

Les identifiants utilisés pour les tests sont des identifiants pédagogiques et ne doivent pas être réutilisés en production.

Les fichiers de configuration publiés sur ce dépôt utilisent des valeurs d'exemple. Aucun mot de passe réel ne doit être versionné.

Une mise en production nécessiterait notamment un renforcement de l'authentification, un filtrage réseau adapté, une gestion des accès et une sécurisation des communications.

---

**Projet réalisé dans le cadre de ma formation en Administration d'Infrastructures Sécurisées (AIS).**
