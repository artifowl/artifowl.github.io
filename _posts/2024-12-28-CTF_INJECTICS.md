---
title: "INJECTICS"
date: 2024-12-28 10:00:00 +0900
categories: [CTF, pentest]
tags: [ctf, pentest, thm, web_auth,medium]
image: 
  path: https://tryhackme-images.s3.amazonaws.com/room-icons/645b19f5d5848d004ab9c9e2-1721317107138
  height: 200
---

# Exploitation des failles d'injection sur une application web vulnérable

## Introduction
Nous nous retrouvons aujourd'hui sur un CTF, qui exploiteras les diverses types de failles d'injections que l'on peut retrouver sur un site WEB.
A travers ce CTF, notre objectif sera de répondre à ces deux questions : 

1. **Quel est la valeur du flag après s'être connecté au panneau administrateur ?**
2. **Quel est le contenu du fichier texte caché dans le dossier flags ?**

L'IP de la machine vulnérable est **10.10.78.73** (cela peut varier en fonction de votre configuration).  
Prêt ? Allons-y !

---

## Étape 1 : Découverte des services
### Scan des ports avec Nmap
Pour commencer, identifions les services qui tournent sur la machine à l'aide de la commande :

```bash
nmap -T4 -A 10.10.78.73
```

**Résultats :**
![Image](https://i.ibb.co/MSpd4gY/1.png){: .normal }

- Port **22** : Service SSH
- Port **80** : Application web (HTTP)

---

## Étape 2 : Exploration de l'application web
### Observation initiale
L'application présente un thème lié aux JO avec un classement des pays par médailles (toutes les cases affichent la même valeur).

![Image](https://i.ibb.co/g377bcb/2.png){: .normal }

#### Points importants trouvés dans le code source de la page :
1. **Notes en commentaire :**
   ```html
   <!-- Website developed by John Tim - dev@injectics.thm -->
   <!-- Mails are stored in mail.log file -->
   ```
  ![Image](https://i.ibb.co/Q9QvV9P/4.png){: .normal }
2. En accédant à `http://10.10.78.73/mail.log`, on trouve des informations sur une fonctionnalité appelée "Injectics". Cette fonctionnalité insère automatiquement des identifiants par défaut dans la table `users` si celle-ci est modifiée.
  ![Image](https://i.ibb.co/Pxdn8nC/5.png){: .normal }

**Identifiants par défaut :**
- Email : `superadmin@injectics.thm` | Password : `superSecurePasswd101`
- Email : `dev@injectics.thm` | Password : `devPasswd123`
Ainsi, supprimer ou modifier des données dans la table user permettrait de se connecter à un de ces deux comptes.

#### Enumération des répertoires
En parrallèle, le lancement de `dirsearch` pour énumérer les répertoires existant à partir d'une liste prédéfini, informe la présence d'une base de donnée SQL mais aussi d'un template engine au seins de l'application : twig.

![Image](https://i.ibb.co/hHRQMGm/18.png){: .normal }

![Image](https://i.ibb.co/Sw9xgwg/19.png){: .normal }


### Découverte des pages accessibles
Les pages `http://10.10.78.73/login.php` et `http://10.10.78.73/adminLogin007.php` contiennent respectivement un formulaire pour se connecter en tant qu'utilisateur et en tant que administrateur.

![Image](https://i.ibb.co/YNtvTzQ/6.png){: .normal}

---

## Étape 3 : Bypass de l'authentification utilisateur
### Analyse de la page login
- Entrée incorrecte : un encadré rouge s'affiche.
- Les caractères `'` et `"` ne sont pas acceptés côté client.

![Image](https://i.ibb.co/R276YkG/8.png){: .normal width="400" height="400" }![Image](https://i.ibb.co/Tq1hgMJ/9.png){: .normal width="400" height="400" }

#### Contournement du filtre client
Malheureusement, le filtre côté client ne suffit pas à protéger le formulaire d'une injection SQL. Ainsi, l'utilisation d'un outil pour intercepter la requete POST pour la modifier après permet de Bypass l'authentification.

**Injection SQL utilisée :**
![Image](https://i.ibb.co/3BTjCcq/10.png){: .normal }

```sql
1' || 1=1 -- -
```
Cette injection renvoie une réponse positive, permettant l'accès à `dashboard.php` en tant que premier utilisateur de la table `users`.

![Image](https://i.ibb.co/xJB1MRG/11.png){: .normal }
---

## Étape 4 : Exploitation des fonctionnalités
### Modification des données
Lorsqu'on clique sur "edit", on est redirigé vers `edit_leaderboard.php`. Une tentative d'injection SQL dans un champ (par exemple : `drop table users`) échoue car les champs exigent des nombres.

![Image](https://i.ibb.co/nDmT1nC/12.png){: .normal }

**Injection SQL ajustée :**

Cela peut s'expliquer parce que la fonction qui permet de faire le calcul du total de médailles, requière un **nombre**.  L'injection doit alors se faire ainsi : `nombre; drop table users -- -`.

```sql
123; drop table users -- -
```
![Image](https://i.ibb.co/KDPXRtR/13.png){: .normal}![Image](https://i.ibb.co/mRnpdNQ/14.png){: .normal }

Cela supprime la table `users` et déclenche la fonctionnalité Injectics, qui réinsère les identifiants par défaut. Nous permettons ensuite d'utiliser `superadmin@injectics.thm` pour accéder au tableau de bord administrateur.

![Image](https://i.ibb.co/2F9dB7H/15.png){: .normal }

---

## Étape 5 : Injection dans le moteur de template Twig
L'accés à une nouvelle page profile est ici alors possible. Modifier notre email, notre nom et notre mot de passe nous est alors proposé. L'entrée d'un nouveau prénom se reflète ici directement dans la page principale :

![Image](https://i.ibb.co/chpHkPQ/16.png){: .normal width="410" height="410" }![Image](https://i.ibb.co/QdsqWCh/17.png){: .normal width="380" height="380" }


#### Test du moteur de template
Un moteur de template est comme une machine qui permet de créer des pages Web de manière dynamique. Voici comment cela fonctionne en termes simples :
Imaginez que vous créez une carte d'anniversaire pour un ami. Vous souhaitez inclure son nom, son âge et un message personnalisé. Au lieu d'écrire une nouvelle carte de toutes pièces, vous utilisez un modèle avec des espaces réservés pour le nom, l'âge et le message.

Malgré le fait qu'ici, rien ne prouve que la page a été généré avec le moteur de template Twig, en insérant une expression comme {% raw %}`{{ 7*7 }}`{% endraw %} dans le champ "Nom", la gestion correcte de l'expression confirme l'utilisation de Twig.

![Image](https://i.ibb.co/4RXhs5K/20.png){: .normal width="400" height="400" } ![Image](https://i.ibb.co/PTcHVzL/21.png){: .normal width="400" height="400" }

### Dépassement du mode sandbox
Cependant, la lecture de fichier avec l'appel de fonction `file_get_contents` ou `fopen` ou encore l'appel simple à des `fonctions PHP` sont sans succées puisque Twig est en mode **sandbox**. Le mode **sandbox** de Twig est une fonctionnalité de sécurité conçue pour restreindre ce que les templates Twig peuvent faire. Un travail de tri, de recherche ainsi que le test manuel de chaque restriction de balise est alors obligatoire.

**Injection Twig réussie :**

{% raw %}
```twig
{{ ['ls /var/www/html/flags', '']|sort('passthru') }}
```
{% endraw %}

Ici, `sort` est un filtre Twig standard qui trie les éléments d'un tableau et l'argument `passthru` est passé comme *callback à sort*. 

`passthru` est une fonction PHP qui exécute une commande système (équivalent à system() mais contrairement à cette dernière *pas filtrée*).


Au final, cette commande liste le contenu du dossier `flags`.

![Image](https://i.ibb.co/6YfZwGG/22.png){: .normal }

![Image](https://i.ibb.co/WK1tg5j/23.png){: .normal }


### Lecture du flag
Pour lire le contenu du fichier de flag :
{% raw %}
```twig
{{ ['cat /var/www/html/flags/5d8af1dc14503c7e4bdc8e51a3469f48.txt', '']|sort('passthru') }}
```
{% endraw %}

**Flag obtenu :**
```
THM{5735172b6c147f4dd649872f73e0fdea}
```




