# TP1 : Containers

## Sommaire

- [TP1 : Containers](#tp1--containers)
- [Sommaire](#sommaire)
- [I. Docker](#i-docker)
  - [1. Install](#1-install)
  - [2. Vérifier l'install](#2-vérifier-linstall)
  - [3. Lancement de conteneurs](#3-lancement-de-conteneurs)
- [II. Images](#ii-images)
  - [Exemple de Dockerfile et utilisation](#exemple-de-dockerfile-et-utilisation)
  - [2. Construisez votre propre Dockerfile](#2-construisez-votre-propre-dockerfile)
- [III. `docker-compose`](#iii-docker-compose)
  - [1. Intro](#1-intro)
  - [2. WikiJS](#2-wikijs)
  - [3. Make your own meow](#3-make-your-own-meow)
- [IV. Docker security](#iv-docker-security)
  - [1. Le groupe docker](#1-le-groupe-docker)
  - [2. Scan de vuln](#2-scan-de-vuln)
  - [3. Petit benchmark secu](#3-petit-benchmark-secu)

## I. Docker

### 1. Install

🌞 **Installer Docker sur la machine**

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

```bash
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

```bash
sudo systemctl --now enable docker
```

```bash
sudo usermod -aG docker $(whoami)
```

### 2. Vérifier l'install

```bash
[dorian@cloud ~]$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
719385e32844: Pull complete 
Digest: sha256:88ec0acaa3ec199d3b7eaf73588f4518c25f9d34f58ce9a0df68429c5af48e8d
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

## 3. Lancement de conteneurs

🌞 **Utiliser la commande `docker run`**

```bash
docker run --name web -m 512m --cpus=0.5 -d -p 9999:7878 -v /home/dorian/nginx/index.html:/var/www/dorian/index.html -v /home/dorian/nginx/default.conf:/etc/nginx/conf.d/toto.conf nginx
```

```bash
[dorian@cloud ~]$ cat nginx/index.html 
Dorian Claverie
```

```bash
[dorian@cloud ~]$ cat nginx/default.conf 
server {
  # on définit le port où NGINX écoute dans le conteneur
  listen 7878;
  
  # on définit le chemin vers la racine web
  # dans ce dossier doit se trouver un fichier index.html
  root /var/www/dorian; 
}
```

## II. Images

La construction d'image avec Docker est basée sur l'utilisation de fichiers `Dockerfile`.

L'idée est la suivante :

- vous créez un dossier de travail
- vous vous déplacez dans ce dossier de travail
- vous créez un fichier `Dockerfile`
  - il contient les instructions pour construire une image
  - `FROM` : indique l'image de base
  - `RUN` : indique des opérations à effectuer dans l'image de base
- vous exécutez une commande `docker build . -t <IMAGE_NAME>`
- une image est produite, visible avec la commande `docker images`

## Exemple de Dockerfile et utilisation

Exemple d'un Dockerfile qui :

- se base sur une image ubuntu
- la met à jour
- installe nginx

```bash
$ cat Dockerfile
FROM ubuntu

RUN apt update -y

RUN apt install -y nginx
```

Une fois ce fichier créé, on peut :

```bash
$ ls
Dockerfile

$ docker build . -t my_own_nginx 

$ docker images

$ docker run -p 8888:80 my_own_nginx nginx -g "daemon off;"

$ curl localhost:8888
$ curl <IP_VM>:8888
```

> La commande `nginx -g "daemon off;"` permet de lancer NGINX au premier-plan, et ainsi demande à notre conteneur d'exécuter le programme NGINX à son lancement.

Plutôt que de préciser à la main à chaque `docker run` quelle commande doit lancer le conteneur (notre `nginx -g "daemon off;"` en fin de ligne ici), on peut, au moment du `build` de l'image, choisir d'indiquer que chaque conteneur lancé à partir de cette image lancera une commande donneé.

Il faut, pour cela, modifier le Dockerfile :

```bash
$ cat Dockerfile
FROM ubuntu

RUN apt update -y

RUN apt install -y nginx

CMD [ "/usr/sbin/nginx", "-g", "daemon off;" ]
```

```bash
$ ls
Dockerfile

$ docker build . -t my_own_nginx

$ docker images

$ docker run -p 8888:80 my_own_nginx

$ curl localhost:8888
$ curl <IP_VM>:8888
```

![Waiting for Docker](./img/waiting_for_docker.jpg)

## 2. Construisez votre propre Dockerfile

🌞 **Construire votre propre image**

- image de base (celle que vous voulez : debian, alpine, ubuntu, etc.)
  - une image du Docker Hub
  - qui ne porte aucune application par défaut
- vous ajouterez
  - mise à jour du système
  - installation de Apache (pour les systèmes debian, le serveur Web apache s'appelle `apache2` et non pas `httpd` comme sur Rocky)
  - page d'accueil Apache HTML personnalisée

➜ Pour vous aider, voilà un fichier de conf minimal pour Apache (à positionner dans `/etc/apache2/apache2.conf`) :

```apache2
# on définit un port sur lequel écouter
Listen 80

# on charge certains modules Apache strictement nécessaires à son bon fonctionnement
LoadModule mpm_event_module "/usr/lib/apache2/modules/mod_mpm_event.so"
LoadModule dir_module "/usr/lib/apache2/modules/mod_dir.so"
LoadModule authz_core_module "/usr/lib/apache2/modules/mod_authz_core.so"

# on indique le nom du fichier HTML à charger par défaut
DirectoryIndex index.html
# on indique le chemin où se trouve notre site
DocumentRoot "/var/www/html/"

# quelques paramètres pour les logs
ErrorLog "logs/error.log"
LogLevel warn
```

➜ Et aussi, la commande pour lancer Apache à la main sur un système Debian par exemple c'est : `apache2 -DFOREGROUND`.

📁 **`Dockerfile`**

# III. `docker-compose`

## 1. Intro

`docker compose` est un outil qui permet de lancer plusieurs conteneurs en une seule commande.

> En plus d'être pratique, il fournit des fonctionnalités additionnelles, liés au fait qu'il s'occupe à lui tout seul de lancer tous les conteneurs. On peut par exemple demander à un conteneur de ne s'allumer que lorsqu'un autre conteneur est devenu "healthy". Idéal pour lancer une application après sa base de données par exemple.

Le principe de fonctionnement de `docker compose` :

- on écrit un fichier qui décrit les conteneurs voulus
  - c'est le `docker-compose.yml`
  - tout ce que vous écriviez sur la ligne `docker run` peut être écrit sous la forme d'un `docker-compose.yml`
- on se déplace dans le dossier qui contient le `docker-compose.yml`
- on peut utiliser les commandes `docker compose` :

```bash
# Allumer les conteneurs définis dans le docker-compose.yml
$ docker compose up
$ docker compose up -d

# Eteindre
$ docker compose down

# Explorer un peu le help, il y a d'autres commandes utiles
$ docker compose --help
```

La syntaxe du fichier peut par exemple ressembler à :

```yml
version: "3.8"

services:
  db:
    image: mysql:5.7
    restart: always
    ports:
      - '3306:3306'
    volumes:
      - "./db/mysql_files:/var/lib/mysql"
    environment:
      MYSQL_ROOT_PASSWORD: beep
      MYSQL_DATABASE: bip
      MYSQL_USER: bap
      MYSQL_PASSWORD: boop

  nginx:
    image: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    restart: unless-stopped
```

> Pour connaître les variables d'environnement qu'on peut passer à un conteneur, comme `MYSQL_ROOT_PASSWORD` au dessus, il faut se rendre sur la doc de l'image en question, sur le Docker Hub par exemple.

## 2. WikiJS

WikiJS est une application web plutôt cool qui comme son nom l'indique permet d'héberger un ou plusieurs wikis. Même principe qu'un MediaWiki donc (solution opensource utilisée par Wikipedia par exemple) mais avec un look plus moderne.

🌞 **Installez un WikiJS** en utilisant Docker

- WikiJS a besoin d'une base de données pour fonctionner
- il faudra donc deux conteneurs : un pour WikiJS et un pour la base de données
- référez-vous à la doc officielle de WikiJS, c'est tout guidé

🌞 **Construisez vous-mêmes l'image de WikiJS**

- récupérez le Dockerfile officiel et utilisez-le pour build localement l'image
- intégrez un build automatique de l'image dans le `docker-compose.yml` (ça se fait avec la clause `build:`)
- assurez-vous, pour des raisons de sécurité, qu'un autre utilisateur que `root` est utilisé

## 3. Make your own meow

Pour cette partie, vous utiliserez une application à vous que vous avez sous la main.

N'importe quelle app fera le taff, un truc dév en cours, en temps perso, au taff, trouvée sur internet, une app web, un serveur de jeu. Peu importe. Ce serait quand même plus cool s'il y a besoin d'au moins deux conteneurs histoire d'exploiter un peu `docker compose`.

Peu importe le langage aussi ! Go, Python, PHP (désolé des gros mots), NodeJS (j'ai déjà dit désolé pour les gros mots ?), ou autres.

🌞 **Conteneurisez votre application**

- créer un `Dockerfile` maison qui porte l'application
- créer un `docker-compose.yml` qui permet de lancer votre application
- vous préciserez dans le rendu les instructions pour lancer l'application
  - indiquer la commande `git clone` pour récupérer votre dépôt
  - le `cd` dans le bon dossier
  - la commande `docker build` pour build l'image
  - la commande `docker-compose` pour lancer le(s) conteneur(s)
  - comme un vrai README qui m'explique comment lancer votre app !

📁 📁 `app/Dockerfile` et `app/docker-compose.yml`. Je veux un sous-dossier `app/` sur votre dépôt git avec ces deux fichiers dedans :)

# IV. Docker security

Dans cette partie, on va survoler quelques aspects de Docker en terme de sécurité.

## 1. Le groupe docker

Si vous avez correctement ajouté votre utilisateur au groupe `docker`, vous utilisez normalement Docker sans taper aucune commande `sudo`.

> La raison technique à ça c'est que vous communiquez avec Docker en utilisant le socket `/var/run/docker.sock`. Demandez-moi si vous voulez + de détails sur ça.

Cela découle sur le fait que vous avez les droits `root` sur la machine. Sans utiliser aucune commande `sudo`, sans devenir `root`, sans même connaître son mot de passe ni rien, si vous êtes membres du groupe `docker` vous pouvez devenir `root` sur la machine.

🌞 **Prouvez que vous pouvez devenir `root`**

- en étant membre du groupe `docker`
- sans taper aucune commande `sudo` ou `su` ou ce genre de choses
- normalement, une seule commande `docker run` suffit
- pour prouver que vous êtes `root`, plein de moyens possibles
  - par exemple un `cat /etc/shadow` qui contient les hash des mots de passe de la machine hôte

## 2. Scan de vuln

Il existe des outils dédiés au scan de vulnérabilités dans des images Docker.

C'est le cas de [Trivy](https://github.com/aquasecurity/trivy) par exemple.

🌞 **Utilisez Trivy**

- effectuez un scan de vulnérabilités sur des images précédemment mises en oeuvre :
  - celle de WikiJS que vous avez build
  - celle de sa base de données
  - l'image de Apache que vous avez build
  - l'image de NGINX officielle utilisée dans la première partie

## 3. Petit benchmark secu

Il existe plusieurs référentiels pour sécuriser une machine donnée qui utilise un OS donné. Un savoir particulièrement recherché pour renforcer la sécurité des serveurs surtout.

Un des référentiels réputé et disponible en libre accès, ce sont [les benchmarks de CIS](https://www.cisecurity.org/cis-benchmarks). Ce sont ni plus ni moins que des guides complets pour sécuriser de façon assez forte une machine qui tourne par exemple sous Debian, Rocky Linux ou bien d'autres.

[Docker développe un petit outil](https://github.com/docker/docker-bench-security) qui permet de vérifier si votre utilisation de Docker est compatible avec les recommandations de CIS.

🌞 **Utilisez l'outil Docker Bench for Security**

- rien à me mettre en rendu, je vous laisse exprimer votre curiosité quant aux résultats
- ce genre d'outils est cool d'un point de vue pédagogique : chaque check que fait le script c'est un truc à savoir finalement !