# Authentification à Deux Facteurs (2FA) avec PAM et Google Authenticator

Guide complet pour sécuriser votre serveur Linux avec l'authentification à deux facteurs utilisant Google Authenticator et les modules PAM.

## Table des Matières

1. [Introduction](#introduction)
2. [Installation de Google Authenticator](#installation-de-google-authenticator)
3. [Configuration de Google Authenticator pour un utilisateur](#configuration-de-google-authenticator-pour-un-utilisateur)
4. [Configuration de PAM pour SSH](#configuration-de-pam-pour-ssh)
5. [Configuration de SSH pour exiger l'authentification à deux facteurs](#configuration-de-ssh-pour-exiger-lauthentification-à-deux-facteurs)
6. [Test de la connexion SSH avec 2FA](#test-de-la-connexion-ssh-avec-2fa)
7. [Bonus : Configuration des exigences de mot de passe avec PAM](#bonus--configuration-des-exigences-de-mot-de-passe-avec-pam)

---

## Introduction

L'authentification à deux facteurs (2FA) ajoute une couche de sécurité supplémentaire en exigeant non seulement un mot de passe ou une clé SSH, mais également un code temporaire généré par l'application Google Authenticator sur votre smartphone.

> **Prérequis** : Ce guide suppose que vous avez déjà configuré l'authentification par clé SSH (voir le guide SSH précédent).

---

## Installation de Google Authenticator

### Installer le module PAM pour Google Authenticator

```bash
# Mettre à jour les paquets
sudo apt update

# Installer le module PAM Google Authenticator
sudo apt install -y libpam-google-authenticator
```

### Installer l'application mobile

- Télécharger l'application **Google Authenticator** sur votre smartphone (iOS/Android)
- Vous pouvez aussi utiliser des alternatives comme **Authy** ou **Aegis** (open source)

---

## Configuration de Google Authenticator pour un utilisateur

### Créer le secret pour l'utilisateur

```bash
# Se connecter en tant que l'utilisateur pour lequel vous souhaitez configurer 2FA
# Si vous êtes root, utilisez :
su - nom_utilisateur

# OU exécuter directement en tant qu'utilisateur
sudo -u nom_utilisateur google-authenticator
```

### Répondre aux questions de configuration

```
# Voulez-vous que les jetons d'authentification soient basés sur le temps ? (O/n)
O

# Voulez-vous mettre à jour le fichier ~/.google_authenticator ? (O/n)
O

# Voulez-vous interdire l'utilisation de plusieurs jetons ? (O/n)
O

# Par défaut, les jetons sont valides 30 secondes. Voulez-vous changer ce délai ? (O/n)
N

# Si le décalage horaire est un problème, voulez-vous faire un calcul de décalage ? (O/n)
N
```

### Sauvegarder le code QR et les codes de secours

> **IMPORTANT** : Notez le code secret affiché et les **codes de secours** (emergency codes). Ces codes vous permettront d'accéder au serveur si vous perdez votre téléphone.

Exemple de sortie :

```
Your new secret key is: XXXXXXXXXXXXXXXX
Your verification code is: 123456
Your emergency codes are:
  12345678
  87654321
  ...

# Scanner le QR code avec l'application Google Authenticator
```

---

## Configuration de PAM pour SSH

### Éditer le fichier de configuration PAM pour SSH

```bash
sudo nano /etc/pam.d/sshd
```

### Ajouter l'authentification Google Authenticator

```bash
# Ajouter cette ligne pour activer Google Authenticator
auth required pam_google_authenticator.so
```

### Optionnel : Utiliser nullok pour les utilisateurs sans 2FA

```bash
# Pour permettre aux utilisateurs sans 2FA de se connecter temporairement
# (à utiliser avec précaution)
auth required pam_google_authenticator.so nullok
```

### Désactiver le common-auth pour éviter les conflits

```bash
# Commenter ou supprimer la ligne suivante (si présente)
# @include common-auth
```

### Exemple de fichier /etc/pam.d/sshd configuré

```bash
# PAM configuration for SSH
@include common-auth
auth required pam_google_authenticator.so
auth required pam_permit.so
account required pam_nologin.so
@include common-account
@include common-session
```

---

## Configuration de SSH pour exiger l'authentification à deux facteurs

### Éditer le fichier de configuration SSH

```bash
sudo nano /etc/ssh/sshd_config
```

### Modifier les paramètres d'authentification

```bash
# Activer l'authentification par défi-réponse
KbdInteractiveAuthentication yes

# Activer l'authentification par clé publique
PubkeyAuthentication yes

# Spécifier les méthodes d'authentification (clé + 2FA)
AuthenticationMethods publickey,keyboard-interactive
```

### Redémarrer le service SSH

```bash
# Redémarrer le service SSH pour appliquer les changements
sudo systemctl restart ssh
```

---

## Test de la connexion SSH avec 2FA

### Vérifier la configuration

```bash
# Vérifier la syntaxe du fichier de configuration SSH
sudo sshd -t
```

### Tester la connexion

```bash
# Se connecter en SSH avec 2FA
# Vous serez invité à entrer le code généré par l'application Google Authenticator
ssh -o PubkeyAuthentication=yes root@10.0.0.100
```

### Exemple de processus de connexion

```
Authentification par clé publique : OK
Verification code: 123456
Welcome to Ubuntu 24.04 LTS
```

### Dépannage

```bash
# Vérifier les logs d'authentification
sudo tail -f /var/log/auth.log

# Si les codes ne fonctionnent pas, vérifier le décalage horaire
# Le serveur et le téléphone doivent être synchronisés
timedatectl
```

---

## Bonus : Configuration des exigences de mot de passe avec PAM

### Installer le paquet pour la gestion des mots de passe

```bash
sudo apt install -y libpam-pwquality
```

### Configurer les exigences de mot de passe

```bash
sudo nano /etc/security/pwquality.conf
```

### Exemple de configuration

```bash
# Longueur minimale du mot de passe (12 caractères)
minlen = 12

# Nombre minimum de classes de caractères (chiffres, majuscules, minuscules, spéciaux)
minclass = 4

# Exiger au moins un chiffre
dcredit = -1

# Exiger au moins une lettre majuscule
ucredit = -1

# Exiger au moins une lettre minuscule
lcredit = -1

# Exiger au moins un caractère spécial
ocredit = -1

# Interdire les mots de passe communs
dictcheck = 1

# Nombre minimum de caractères différents du mot de passe précédent
difok = 4
```

### Modifier la configuration PAM pour les mots de passe

```bash
sudo nano /etc/pam.d/common-password
```

### Ajouter les exigences de mot de passe

```bash
# Trouver la ligne contenant pam_pwquality.so et la modifier :
password requisite pam_pwquality.so retry=3 minlen=12 minclass=4 difok=4
```

### Tester la création d'un nouvel utilisateur

```bash
# Créer un nouvel utilisateur
sudo adduser testuser

# Vous serez invité à entrer un mot de passe respectant les exigences définies
# Si le mot de passe ne répond pas aux critères, il sera rejeté
```

### Exemples de messages d'erreur

```
BAD PASSWORD: it is too short
BAD PASSWORD: it is too simplistic and system
BAD PASSWORD: it is based on a dictionary word
```

---

## Résumé des fichiers modifiés

| Fichier | Description |
|---------|-------------|
| `/etc/pam.d/sshd` | Configuration PAM pour SSH avec Google Authenticator |
| `/etc/ssh/sshd_config` | Configuration SSH pour l'authentification à deux facteurs |
| `/etc/security/pwquality.conf` | Configuration des exigences de mot de passe |
| `/etc/pam.d/common-password` | Activation des exigences de mot de passe |

