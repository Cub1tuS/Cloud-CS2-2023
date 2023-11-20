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

🌞 **Construire votre propre image**

```bash
[dorian@laptop-dorian cloud]$ cat Dockerfile 
FROM ubuntu

RUN apt update -y

RUN apt install -y apache2

RUN echo "hello dorian" > /var/www/html/index.html 

COPY apache2.conf /etc/apache2/apache2.conf

CMD [ "apache2", "-D", "FOREGROUND" ]
```

- Conf apache

```bash
[dorian@laptop-dorian cloud]$ cat apache2.conf 
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
ErrorLog "/var/log/apache2/error.log"
LogLevel warn
```

```bash
docker build . -t mon
```

```bash
docker run -d -p 8888:80 mon
```

```bash
[dorian@laptop-dorian cloud]$ curl localhost:8888
hello dorian
```

## III. `docker-compose`

🌞 **Installez un WikiJS** en utilisant Docker

```bash
[dorian@laptop-dorian cloud]$ nano docker-compose.yml
```

```bash
[dorian@laptop-dorian cloud]$ cat docker-compose.yml
version: "3"
services:

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: wiki
      POSTGRES_PASSWORD: wikijsrocks
      POSTGRES_USER: wikijs
    logging:
      driver: "none"
    restart: unless-stopped
    volumes:
      - db-data:/var/lib/postgresql/data

  wiki:
    image: ghcr.io/requarks/wiki:2
    depends_on:
      - db
    environment:
      DB_TYPE: postgres
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: wikijs
      DB_PASS: wikijsrocks
      DB_NAME: wiki
    restart: unless-stopped
    ports:
      - "80:3000"

volumes:
  db-data:
```

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

## IV. Docker security

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