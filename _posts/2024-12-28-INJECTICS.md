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
![Image](https://media.discordapp.net/attachments/1313520626299961344/1326536102038732874/image.png?ex=67831444&is=6781c2c4&hm=fc4acfbfe7cbf1a2f1963eb5e1006828eb32c5bf5265550bf2f7a4a8cf5e568e&=&format=webp&quality=lossless&width=1440&height=500){: .normal }

- Port **22** : Service SSH
- Port **80** : Application web (HTTP)

---

## Étape 2 : Exploration de l'application web
### Observation initiale
L'application présente un thème lié aux JO avec un classement des pays par médailles (toutes les cases affichent la même valeur).

![Image](https://media.discordapp.net/attachments/1313520626299961344/1326537453900791859/image.png?ex=67831586&is=6781c406&hm=122e52bb7a496cdd1230da1037c38ff8c2454f0a8207615711daa811dc3c3c10&=&format=webp&quality=lossless&width=687&height=361){: .normal }

#### Points importants trouvés dans le code source de la page :
1. **Notes en commentaire :**
   ```html
   <!-- Website developed by John Tim - dev@injectics.thm -->
   <!-- Mails are stored in mail.log file -->
   ```
  ![Image](https://media.discordapp.net/attachments/1313520626299961344/1326538359354691584/image.png?ex=6783165e&is=6781c4de&hm=ff82118b137917bd5113364da5bd13c56b657fc4b747f47da9a15939ba901f16&=&format=webp&quality=lossless&width=782&height=515){: .normal }
2. En accédant à `http://10.10.78.73/mail.log`, on trouve des informations sur une fonctionnalité appelée "Injectics". Cette fonctionnalité insère automatiquement des identifiants par défaut dans la table `users` si celle-ci est modifiée.
  ![Image](https://media.discordapp.net/attachments/1313520626299961344/1326539492944908299/image.png?ex=6783176d&is=6781c5ed&hm=7b99dc0ea7454c2a8cdfa49444783c34935e20940010e9fe120e735b7fa07982&=&format=webp&quality=lossless&width=1440&height=370){: .normal }

**Identifiants par défaut :**
- Email : `superadmin@injectics.thm` | Password : `superSecurePasswd101`
- Email : `dev@injectics.thm` | Password : `devPasswd123`
Ainsi, supprimer ou modifier des données dans la table user permettrait de se connecter à un de ces deux comptes.

#### Enumération des répertoires
En parrallèle, le lancement de `dirsearch` pour énumérer les répertoires existant à partir d'une liste prédéfini, informe la présence d'une base de donnée SQL mais aussi d'un template engine au seins de l'application : twig.

![Image](https://media.discordapp.net/attachments/1313520626299961344/1326817644900515922/image.png?ex=6782c8f9&is=67817779&hm=b0dcc73e8cb576e875b98b378215ec1a86a3d2bac7b22f16455915d96ff0e5b8&=&format=webp&quality=lossless&width=900&height=662){: .normal }

![Image](https://media.discordapp.net/attachments/1313520626299961344/1326817736373829643/image.png?ex=6782c90f&is=6781778f&hm=8044501a258f39c8ffbb37c554e24cb33f28c1be934a1f0112b77ea94a9e1b62&=&format=webp&quality=lossless&width=540&height=437){: .normal }


### Découverte des pages accessibles
Les pages `http://10.10.78.73/login.php` et `http://10.10.78.73/adminLogin007.php` contiennent respectivement un formulaire pour se connecter en tant qu'utilisateur et en tant que administrateur.

![Image](https://media.discordapp.net/attachments/1313520626299961344/1326541979609333845/image.png?ex=678319bd&is=6781c83d&hm=09477d3e8246e5c4710a1a2f6675210052da247728048a287c391608aeead3c4&=&format=webp&quality=lossless&width=1316&height=662){: .normal width="380" height="380" } ![Image](https://media.discordapp.net/attachments/1313520626299961344/1326541980121174026/image.png?ex=678319be&is=6781c83e&hm=be12f5f23832820afccde3d16d1b4a9b1487621e90ca9c36819db71b2f38692b&=&format=webp&quality=lossless&width=1395&height=662){: .normal width="400" height="400" }

---

## Étape 3 : Bypass de l'authentification utilisateur
### Analyse de la page login
- Entrée incorrecte : un encadré rouge s'affiche.
- Les caractères `'` et `"` ne sont pas acceptés côté client.

![Image](https://media.discordapp.net/attachments/1313520626299961344/1326543094069461052/image.png?ex=67831ac7&is=6781c947&hm=3bea6ec3690b07fa805d86003f301596ca2ab8ce07495316c5f2eecb2e8e8152&=&format=webp&quality=lossless&width=651&height=375){: .normal width="380" height="380" } ![Image](https://media.discordapp.net/attachments/1313520626299961344/1326543094317060127/image.png?ex=67831ac7&is=6781c947&hm=368769b4a19582e23e245dc734758c915f33e20842e88d24e03eed24a6aba03c&=&format=webp&quality=lossless&width=887&height=520){: .normal width="400" height="400" }

#### Contournement du filtre client
Malheureusement, le filtre côté client ne suffit pas à protéger le formulaire d'une injection SQL. Ainsi, l'utilisation d'un outil pour intercepter la requete POST pour la modifier après permet de Bypass l'authentification.

**Injection SQL utilisée :**
![Image](https://media.discordapp.net/attachments/1313520626299961344/1326559304790048860/image.png?ex=67828120&is=67812fa0&hm=2af10caa030cd718fe4ffc04a5a56bf8aff9770fb2f3ed544b88da02b9c14c1c&=&format=webp&quality=lossless&width=1296&height=662){: .normal }

```sql
1' || 1=1 -- -
```
Cette injection renvoie une réponse positive, permettant l'accès à `dashboard.php` en tant que premier utilisateur de la table `users`.

![Image](https://media.discordapp.net/attachments/1313520626299961344/1326561954957099110/image.png?ex=67828398&is=67813218&hm=0ddd5554b44f2122480056981d832d15d8092208d68097feb30f763a0e28a692&=&format=webp&quality=lossless&width=1261&height=662){: .normal }
---

## Étape 4 : Exploitation des fonctionnalités
### Modification des données
Lorsqu'on clique sur "edit", on est redirigé vers `edit_leaderboard.php`. Une tentative d'injection SQL dans un champ (par exemple : `drop table users`) échoue car les champs exigent des nombres.

![Image](https://media.discordapp.net/attachments/1313520626299961344/1326806782437490699/image.png?ex=6782bedb&is=67816d5b&hm=5c26c2edcb8c4c05ab9909e21378df6cbe01e395988311431d89dc196a5a12d1&=&format=webp&quality=lossless&width=1440&height=570){: .normal }

**Injection SQL ajustée :**

Cela peut s'expliquer parce que la fonction qui permet de faire le calcul du total de médailles, requière un **nombre**.  L'injection doit alors se faire ainsi : `nombre; drop table users -- -`.

```sql
123; drop table users -- -
```
![Image](https://media.discordapp.net/attachments/1313520626299961344/1326807702785232916/image.png?ex=6782bfb7&is=67816e37&hm=7db25f4377194aeab3e6c759ee34bd5aeab9f1b0b1b1dada453676b14bc4116f&=&format=webp&quality=lossless&width=1440&height=572){: .normal width="430" height="430" } ![Image](https://media.discordapp.net/attachments/1313520626299961344/1326807703217115248/image.png?ex=6782bfb7&is=67816e37&hm=04b9979936aa2aa7e6ebaf56e94f544a3ad9c96fce50d64fbd59050d466a6e2a&=&format=webp&quality=lossless&width=1426&height=662){: .normal width="380" height="380" }

Cela supprime la table `users` et déclenche la fonctionnalité Injectics, qui réinsère les identifiants par défaut. Nous permettons ensuite d'utiliser `superadmin@injectics.thm` pour accéder au tableau de bord administrateur.

![Image](https://media.discordapp.net/attachments/1313520626299961344/1326808696331829258/image.png?ex=6782c0a4&is=67816f24&hm=d1e2a21d5d36257ba09620483ab06b3aba33b2f07ef0a97cad1e0f08792f143a&=&format=webp&quality=lossless&width=1267&height=662){: .normal }

---

## Étape 5 : Injection dans le moteur de template Twig
L'accés à une nouvelle page profile est ici alors possible. Modifier notre email, notre nom et notre mot de passe nous est alors proposé. L'entrée d'un nouveau prénom se reflète ici directement dans la page principale :

![Image](https://media.discordapp.net/attachments/1313520626299961344/1326813811482562622/image.png?ex=6782c567&is=678173e7&hm=acb396b7000778bfbbff6a1b118118ec314f8a38641b1bd740974a4f52c5188c&=&format=webp&quality=lossless&width=1302&height=662){: .normal width="430" height="430" } ![Image](https://media.discordapp.net/attachments/1313520626299961344/1326813811713380392/image.png?ex=6782c567&is=678173e7&hm=8fc68c4cf026c28be49ec448c90cfc0899a8c2e74b61afa96fb8a48279f58ab9&=&format=webp&quality=lossless&width=1170&height=662){: .normal width="380" height="380" }


#### Test du moteur de template
Un moteur de template est comme une machine qui permet de créer des pages Web de manière dynamique. Voici comment cela fonctionne en termes simples :
Imaginez que vous créez une carte d'anniversaire pour un ami. Vous souhaitez inclure son nom, son âge et un message personnalisé. Au lieu d'écrire une nouvelle carte de toutes pièces, vous utilisez un modèle avec des espaces réservés pour le nom, l'âge et le message.

Malgré le fait qu'ici, rien ne prouve que la page a été généré avec le moteur de template Twig, en insérant une expression comme {% raw %}`{{ 7*7 }}`{% endraw %} dans le champ "Nom", la gestion correcte de l'expression confirme l'utilisation de Twig.

![Image](https://media.discordapp.net/attachments/1313520626299961344/1326819179277582406/image.png?ex=6782ca67&is=678178e7&hm=5cab8441e33f563a9913421ada12c27c71348b79698a1532f0b9be9994e52b20&=&format=webp&quality=lossless&width=818&height=662){: .normal width="400" height="400" } ![Image](https://media.discordapp.net/attachments/1313520626299961344/1326819179529109568/image.png?ex=6782ca67&is=678178e7&hm=4f6ca17463e16af615e37350fa100ee6b82877c27d57ac10590487e4e0a99f62&=&format=webp&quality=lossless&width=835&height=662){: .normal width="400" height="400" }

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

![Image](https://media.discordapp.net/attachments/1313520626299961344/1326837408351911988/image.png?ex=6782db61&is=678189e1&hm=96b6e3f5add5319c9006f2485578ff6ece8ead22c2b93c0fe101a7629ea90026&=&format=webp&quality=lossless&width=1160&height=662){: .normal }

![Image](https://media.discordapp.net/attachments/1313520626299961344/1326837408662163467/image.png?ex=6782db61&is=678189e1&hm=f6a52be5eea0088b62543c9e9c0074f07cbf408822278e844bfe4e29b41ed8dd&=&format=webp&quality=lossless&width=718&height=662){: .normal }


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




