---
title: "SILVER PLATTER - Walkthrough"
date: 2025-01-28 10:00:00 +0900
categories: [CTF, Pentest]
tags: [ctf, pentest, thm, medium]
image:
  path: https://tryhackme-images.s3.amazonaws.com/room-icons/5f9c7574e201fe31dad228fc-1718362240227
  height: 200
---

## Introduction
Aujourd'hui, nous allons explorer un CTF (Capture The Flag) dont l'objectif est de compromettre un serveur web et de récupérer deux drapeaux (flags) cachés. Les questions auxquelles nous devrons répondre sont les suivantes :

1. **What is the user flag?**
2. **What is the root flag?**

Le scénario indique que l'équipe de sécurité a renforcé le serveur contre les attaques courantes. De plus, la politique de mots de passe exige que ceux-ci ne figurent pas dans la liste de mots de passe courants (comme `rockyou.txt`), ce qui rend les attaques par force brute difficiles.

---

## Étape 1 : Découverte des services
### Scan des ports avec Nmap
Pour commencer, nous effectuons un scan agressif des ports avec **Nmap** afin d'identifier les services actifs sur la machine cible.

```bash
nmap -T4 -A 10.10.21.248
```

**Résultats :**

![Image](https://i.ibb.co/F6hvcCh/image.png){: .normal }

- **Port 22** : Service SSH
- **Port 80** : Serveur Web
- **Port 8080** : Un autre serveur web (probablement une redirection ou un service similaire)

---

## Étape 2 : Exploration de l'application web
### Observation initiale
Le site web hébergé sur le port 80 est basé sur un template provenant de [HTML5 UP](https://html5up.net/). Il s'agit d'un site de présentation pour une entreprise fictive nommée **Hack Smarter Security**.

![Image](https://i.ibb.co/DwHKn5P/image.png){: .normal }

Les différentes sections du site révèlent que **Hack Smarter Security** est une communauté de hackers appelée *1337est*, dirigée par un certain **Tyler Ramsbey**. Leur spécialité est le pentest web, et ils semblent utiliser leurs compétences pour extorquer des entreprises. La section **Contact** invite les visiteurs à contacter un utilisateur nommé **scr1ptkiddy** sur **Silverpeas**.

Le site ne semble pas offrir de fonctionnalités interactives ou de vulnérabilités évidentes. Une inspection plus approfondie confirme qu'il s'agit simplement d'un template statique.

![Image](https://i.ibb.co/6X68pr5/image.png){: .normal }

---

### Exploration du port 8080
Le serveur web sur le port 8080 renvoie une **erreur 404**, et une inspection du code source ne révèle aucune information utile. Nous devons donc chercher ailleurs.

![Image](https://i.ibb.co/h9zZT09/image.png){: .normal }

---

### Découverte de Silverpeas
En revenant au site web sur le port 80, nous remarquons une mention de **Silverpeas**, une plateforme collaborative permettant de partager des documents et de gérer des projets. Selon la documentation, Silverpeas est accessible via un intranet/extranet.

```html
   http://localhost:8000/silverpeas
```

![Image](https://i.ibb.co/2jthWxf/image.png){: .normal }

En essayant d'accéder à Silverpeas sur les ports 80 et 8080, nous obtenons respectivement une **une erreur** et **une page de login**.

![Image](https://i.ibb.co/sV1s8zw/image.png){: .normal width="400" height="400" }
![Image](https://i.ibb.co/G470PcRM/image.png){: .normal width="400" height="400" }

---

## Étape 3 : Contournement de l'authentification
### Tentative de réinitialisation de mot de passe
Comme mentionné dans la section **Contact**, l'utilisateur **scr1ptkiddy** semble être un compte valide. En essayant de réinitialiser le mot de passe pour ce compte, nous obtenons un message indiquant que le mot de passe sera envoyé par e-mail.

![Image](https://i.ibb.co/3vLR0d3/image.png){: .normal }

Cela suggère que **scr1ptkiddy** est un utilisateur valide, mais nous ne pouvons pas récupérer le mot de passe directement. Nous devons donc trouver un autre moyen de contourner l'authentification.

---

### Exploitation de CVE-2024-36042
En recherchant des vulnérabilités dans Silverpeas, nous découvrons une faille d'authentification (**CVE-2024-36042**). Cette vulnérabilité permet de contourner l'authentification en supprimant le paramètre `Password` lors de la requête de connexion.

En exploitant cette faille, nous parvenons à accéder à l'intranet de **scr1ptkiddy** sans fournir de mot de passe.

![Image](https://i.ibb.co/KX2JnfV/image.png){: .normal }
![Image](https://i.ibb.co/JRyRcwn/image.png){: .normal }

---

## Étape 4 : Accès à l'administration de Silverpeas
### Accès au compte SilverAdmin
Selon la documentation de Silverpeas, les identifiants par défaut pour l'administrateur sont **SilverAdmin/SilverAdmin**. En utilisant la même technique de contournement, nous accédons au compte **SilverAdmin**.

![Image](https://i.ibb.co/PYGGy5Z/image.png){: .normal }
![Image](https://i.ibb.co/LgCqTZr/image.png){: .normal }

---

### Exploration des outils administratifs
Une fois connecté en tant qu'administrateur, nous explorons les outils disponibles. Deux outils semblent prometteurs :

1. **Silver Crawling** : Permet de parcourir les fichiers et répertoires du serveur.
2. **Web Site's Designer** : Permet de créer des pages web, ce qui pourrait être utilisé pour uploader un reverse shell.

![Image](https://i.ibb.co/ksYPRbSR/image.png){: .normal width="400" height="400" }![Image](https://i.ibb.co/x8YdX61C/image.png){: .normal width="400" height="400" }

Cependant, ces outils ne donnent pas les résultats escomptés. Bien que **Silver Crawling** permette d'accéder à des fichiers sensibles comme `/etc/shadow` et `/etc/passwd`, ces fichiers ne contiennent pas de mots de passe configurés pour les utilisateurs. De plus, le répertoire `home` est vide.

L'outil **Web Site's Designer** ne permet pas non plus d'uploader un reverse shell, car il est limité à la création de liens web.

---

## Étape 5 : Recherche d'informations sensibles
### Exploitation d'une vulnérabilité IDOR
En explorant les notifications de l'administrateur, nous remarquons que l'URL de la notification contient un paramètre `ID`. En modifiant ce paramètre, nous pouvons accéder aux messages d'autres utilisateurs, exploitant ainsi une vulnérabilité de type **IDOR (Insecure Direct Object Reference)**.

En parcourant les messages, nous trouvons un message à l'`ID 6` contenant des identifiants SSH :

- **Username** : *tim*
- **Password** : *cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol*

![Image](https://i.ibb.co/hxFMcDPM/image.png){: .normal width="400" height="400" }
![Image](https://i.ibb.co/Mk58fNVR/image.png){: .normal width="400" height="400" }

---

### Récupération du user flag
En utilisant ces identifiants, nous nous connectons en SSH à l'utilisateur **tim** et récupérons le **user flag** dans son répertoire home.

![Image](https://i.ibb.co/QskRhtG/image.png){: .normal }

---

## Étape 6 : Escalade de privilèges
### Analyse des logs système
En vérifiant les groupes auxquels appartient l'utilisateur **tim**, nous constatons qu'il fait partie du groupe **admin**, ce qui lui permet d'accéder aux logs système. En inspectant les fichiers `/var/log/auth.*`, nous découvrons que l'utilisateur **tyler** s'est connecté à **PostgreSQL** avec le mot de passe **_Zd_zx7N823/**.

![Image](https://i.ibb.co/BVKc9gYv/image.png){: .normal }

---

### Accès à l'utilisateur tyler
En supposant que **tyler** utilise le même mot de passe pour son compte SSH, nous tentons de nous connecter avec les identifiants suivants :

- **Username** : *tyler*
- **Password** : *_Zd_zx7N823/*

La connexion est réussie, et nous obtenons un accès SSH à l'utilisateur **tyler**.

![Image](https://i.ibb.co/d0jbdgpN/image.png){: .normal }

---

### Accès root
L'utilisateur **tyler** appartient au groupe **sudo**, ce qui lui permet d'exécuter des commandes en tant que superutilisateur. En utilisant la commande `sudo -l`, nous confirmons que **tyler** peut exécuter toutes les commandes en tant que **root**.

Nous récupérons alors le **root flag** dans le répertoire home de **root**.

![Image](https://i.ibb.co/hJ8GxSH1/image.png){: .normal }

---

## Conclusion
Ce CTF nous a permis d'explorer plusieurs techniques d'exploitation, notamment le contournement de l'authentification, l'exploitation de vulnérabilités IDOR, et l'escalade de privilèges via l'accès SSH. Ces étapes illustrent l'importance de sécuriser les applications web et de surveiller les accès aux systèmes sensibles.

---

J'espère que ce walkthrough vous a été utile.
