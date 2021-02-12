## Contents

Comment installer Prometheus sur Ubuntu 16.04

* Conditions préalables
* Étape 1 - Création d'utilisateurs de service
* Étape 2 - Téléchargement de Prometheus
* Étape 3 - Configuration de Prometheus
* Étape 4 - Exécution de Prometheus
* Étape 5 - Téléchargement de Node Exporter
* Étape 6 - Exécution de Node Exporter
* Étape 7 - Configuration de Prometheus pour exporter l'exportateur de nœuds
* Étape 8 - Sécurisation de Prometheus
* Étape 9 - Test de Prometheus
* Conclusion

## Introduction

[Prometheus](https://prometheus.io/) est un puissant système de
surveillance open source qui collecte les métriques de vos services et
les stocke dans une base de données chronologique. Il offre un modèle de
données multidimensionnel, un langage de requête flexible et diverses
possibilités de visualisation grâce à des outils comme
[Grafana](https://grafana.com/).

Par défaut, Prometheus exporte uniquement des métriques sur lui-même
(par exemple le nombre de requêtes reçues, sa consommation de mémoire,
etc.). Mais vous pouvez étendre considérablement Prometheus en
installant des *exportateurs*, des programmes optionnels qui génèrent
des métriques supplémentaires.

Les exportateurs - à la fois ceux officiels gérés par l'équipe
Prometheus et ceux fournis par la communauté - fournissent des
informations sur tout, de l'infrastructure, des bases de données et des
serveurs Web aux systèmes de messagerie, aux API, etc.

Certains des choix les plus populaires incluent:

* [node_exporter](https://github.com/prometheus/node_exporter) : Cela
    produit des métriques sur l'infrastructure, y compris l'utilisation
    actuelle du processeur, de la mémoire et du disque, ainsi que des
    statistiques d'E/S et du réseau, telles que le nombre d'octets lus
    sur un disque ou la charge moyenne d'un serveur.

* [blackbox_exporter](https://github.com/prometheus/blackbox_exporter) :
    Cela génère des métriques dérivées de protocoles de détection tels
    que HTTP et HTTPS pour déterminer la disponibilité des points de
    terminaison, le temps de réponse, etc.

* [mysqld_exporter](https://github.com/prometheus/mysqld_exporter) :
    Il rassemble des métriques liées à un serveur MySQL, telles que le
    nombre de requêtes exécutées, le temps de réponse moyen des requêtes
    et l'état de réplication du cluster.

* [rabbitmq_exporter](https://github.com/kbudde/rabbitmq_exporter) :
    Ceci génère des métriques sur le système de messagerie
    [RabbitMQ](https://www.rabbitmq.com/), y compris le nombre de
    messages publiés, le nombre de messages prêts à être livrés et la
    taille de tous les messages de la file d'attente.

* [nginx-vts-exporter](https://github.com/hnlq715/nginx-vts-exporter) :
    Cela fournit des métriques sur un serveur Web Nginx utilisant le
    [module Nginx VTS](https://github.com/vozlt/nginx-module-vts) , y
    compris le nombre de connexions ouvertes, le nombre de réponses
    envoyées (regroupées par codes de réponse) et la taille totale des
    demandes envoyées ou reçues en octets .

Vous pouvez trouver une liste plus complète des exportateurs officiels
et communautaires sur [le site Web de Prometheus](https://prometheus.io/docs/instrumenting/exporters/) .

Dans ce cours, vous allez installer, configurer et sécuriser
Prometheus et Node Exporter pour générer des métriques qui faciliteront
la surveillance des performances de votre serveur.

## Conditions préalables

Avant de suivre ce cours, assurez-vous d'avoir:

* Une VM Ubuntu 16.04, configurée en suivant le [cours de configuration initiale du serveur avec Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04)
    , y compris un utilisateur sudo non root et un pare-feu.

* Nginx installé en suivant les deux premières étapes du tutoriel
    [Comment installer Nginx sur Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-16-04).

## Étape 1 - Création d'utilisateurs de service

Pour des raisons de sécurité, nous commencerons par créer deux nouveaux
comptes utilisateurs, **prometheus** et **node_exporter**. Nous
utiliserons ces comptes tout au long du cours pour isoler la
propriété des fichiers et répertoires principaux de Prometheus.

Créez ces deux utilisateurs et utilisez les options `--no-create-home` et
`--shell /bin/false` afin que ces utilisateurs ne puissent pas se
connecter au serveur.
```
sudo useradd --no-create-home --shell /bin/false prometheus
sudo useradd --no-create-home --shell /bin/false node_exporter
```
Avant de télécharger les binaires Prometheus,
créez les répertoires nécessaires pour stocker les fichiers et les données de Prometheus. En suivant les conventions Linux standard, nous allons créer un repertoire
sous `/etc` pour les fichiers de configuration de Prometheus et un
répertoire sous `/var/lib` pour ses données.
```
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
```
Maintenant, définissez l'utilisateur et le groupe des nouveaux répertoires à
**prometheus** .
```
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```
Avec nos utilisateurs et répertoires en place, nous pouvons maintenant
télécharger Prometheus puis créer le fichier de configuration minimal
pour exécuter Prometheus pour la première fois.

## Étape 2 - Téléchargement de Prometheus

Tout d'abord, téléchargez et décompressez la version stable actuelle de
Prometheus dans votre répertoire personnel. Vous pouvez trouver les
derniers binaires ainsi que leurs sommes de contrôle sur la
[page de téléchargement de Prometheus](https://prometheus.io/download/) .
```
cd ~
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.24.1/prometheus-2.24.1.linux-amd64.tar.gz
```
Ensuite, utilisez la sha256sum commande pour générer une somme de
contrôle du fichier téléchargé:
```
sha256sum prometheus-2.24.1.linux-amd64.tar.gz
```
Comparez la sortie de cette commande avec la somme de contrôle sur la
page de téléchargement Prometheus pour vous assurer que votre fichier
est à la fois authentique et non corrompu.

Output
```
e12917b25b32980daee0e9cf879d9ec197e2893924bd1574604eb0f550034d46
prometheus-2.0.0.linux-amd64.tar.gz
```
Si les sommes de contrôle ne correspondent pas, supprimez le fichier
téléchargé et répétez les étapes précédentes pour re-télécharger le
fichier.

Maintenant, décompressez l'archive téléchargée.
```
tar xvf prometheus-2.24.1.linux-amd64.tar.gz
```
Cela créera un répertoire appelé prometheus-2.24.1.linux-amd64 contenant
deux fichiers binaires ( prometheus et promtool) et deux répertoires
consoles et des console_libraries contenant les fichiers d'interface
Web, une licence, une notice et plusieurs fichiers d'exemple.

Copiez les deux binaires dans le /usr/local/bin répertoire.
```
sudo cp prometheus-2.24.1.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.24.1.linux-amd64/promtool /usr/local/bin/
```
Définissez la propriété de l'utilisateur et du groupe sur les binaires
sur l' utilisateur **prometheus** créé à l'étape 1.
```
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```
Copiez les répertoires consoles et console_libraries dans
/etc/prometheus.
```
sudo cp -r prometheus-2.24.1.linux-amd64/consoles /etc/prometheus
sudo cp -r prometheus-2.24.1.linux-amd64/console_libraries /etc/prometheus
```
Définissez la propriété de l'utilisateur et du groupe sur les
répertoires sur l' utilisateur **prometheus**. L'utilisation de
l'indicateur –R garantit que la propriété est également définie sur les
fichiers à l'intérieur du répertoire.
```
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```
Enfin, supprimez les fichiers restants de votre répertoire personnel car
ils ne sont plus nécessaires.
```
rm -rf prometheus-2.24.1.linux-amd64.tar.gz prometheus-2.24.1.linux-amd64
```
Maintenant que Prometheus est installé, nous allons créer ses fichiers
de configuration et de service en préparation de sa première exécution.

## Étape 3 - Configuration de Prometheus

Dans le répertoire /etc/prometheus, utilisez nano ou votre éditeur de
texte préféré pour créer un fichier de configuration nommé `prometheus.yml`.
Pour l'instant, ce fichier contiendra juste assez d'informations
pour exécuter Prometheus pour la première fois.
```
sudo nano /etc/prometheus/prometheus.yml
```
**Attention:** le fichier de configuration de Prometheus utilise le
[format YAML](http://www.yaml.org/start.html), qui interdit strictement
les **tabulations** et nécessite **deux espaces pour l'indentation**.
Prometheus ne démarre pas si le fichier de configuration n'est pas
formaté correctement.

Dans les paramètres `global`, définissez l'intervalle par défaut pour les
collectes (scrapes). Notez que Prometheus appliquera ces paramètres à
chaque exportateur à moins que les paramètres propres d'un exportateur
individuel ne remplacent les globaux.

Fichier de configuration Prometheus partie 1 - /etc/prometheus/prometheus.yml
```
global:
  scrape_interval: 15s
```
Cette valeur `scrape_interval` indique à Prometheus de collecter des
métriques de ses exportateurs toutes les 15 secondes, ce qui est
suffisamment long pour la plupart des exportateurs.

Maintenant, ajoutez Prometheus lui-même à la liste des exportateurs à
supprimer avec la directive `scrape_configs` suivante :

Fichier de configuration Prometheus, partie 2 - /etc/prometheus/prometheus.yml
```
...
scrape_configs:
- job_name: 'prometheus'
  scrape_interval: 5s
  static_configs:
  - targets: ['localhost:9090']
```
Prometheus utilise le `job_name` pour étiqueter les exportateurs dans les
requêtes et les graphiques, alors assurez-vous de choisir quelque chose
de descriptif ici.

Et, comme Prometheus exporte des données importantes sur lui-même que
vous pouvez utiliser pour surveiller les performances et le débogage,
nous avons remplacé la directive globale `scrape_interval` de 15 secondes
à 5 secondes pour des mises à jour plus fréquentes.

Enfin, Prometheus utilise les directives `static_configs` et `targets` pour
déterminer où les exportateurs s'exécutent. Étant donné que cet
exportateur particulier est en cours d'exécution sur le même serveur que
Prometheus lui-même, on peut utiliser localhost au lieu d'une adresse
IP ainsi que le port par défaut, 9090.

Votre fichier de configuration devrait maintenant ressembler à ceci:

Fichier de configuration Prometheus - /etc/prometheus/prometheus.yml
```
global:
  scrape_interval: 15s
scrape_configs:
- job_name: 'prometheus'
  scrape_interval: 5s
  static_configs:
  - targets: ['localhost:9090']
```
Enregistrez le fichier et quittez votre éditeur de texte.

Maintenant, définissez la propriété de l'utilisateur et du groupe sur le
fichier de configuration sur l'utilisateur **prometheus** créé à
l'étape 1.
```
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```
Une fois la configuration terminée, nous sommes prêts à tester
Prometheus en l'exécutant pour la première fois.

## Étape 4 - Exécution de Prometheus

Démarrez Prometheus en tant qu'utilisateur **prometheus** , en
fournissant le chemin d'accès au fichier de configuration et au
répertoire de données.
```
sudo -u prometheus /usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/  \
--web.console.templates=/etc/prometheus/consoles  \
--web.console.libraries=/etc/prometheus/console_libraries
```
La sortie contient des informations sur la progression du chargement de
Prometheus, le fichier de configuration et les services associés. Il
confirme également que Prometheus écoute sur le port 9090.

Output
```
level=info ts=2017-11-17T18:37:27.474530094Z caller=main.go:215
msg="Starting Prometheus" version="(version=2.0.0, branch=HEAD,
revision=0a74f98628a0463dddc90528220c94de5032d1a0)"

level=info ts=2017-11-17T18:37:27.474758404Z caller=main.go:216
build_context="(go=go1.9.2, user=root@615b82cb36b6,
date=20171108-07:11:59)"

level=info ts=2017-11-17T18:37:27.474883982Z caller=main.go:217
host_details="(Linux 4.4.0-98-generic #121-Ubuntu SMP Tue Oct 10
14:24:03 UTC 2017 x86_64 prometheus-update (none))"

level=info ts=2017-11-17T18:37:27.483661837Z caller=web.go:380
component=web msg="Start listening for connections" address=0.0.0.0:9090

level=info ts=2017-11-17T18:37:27.489730138Z caller=main.go:314
msg="Starting TSDB"

level=info ts=2017-11-17T18:37:27.516050288Z caller=targetmanager.go:71
component="target manager" msg="Starting target manager..."

level=info ts=2017-11-17T18:37:27.537629169Z caller=main.go:326
msg="TSDB started"

level=info ts=2017-11-17T18:37:27.537896721Z caller=main.go:394
msg="Loading configuration file" filename=/etc/prometheus/prometheus.yml

level=info ts=2017-11-17T18:37:27.53890004Z caller=main.go:371
msg="Server is ready to receive requests."
```
Si vous obtenez un message d'erreur, vérifiez que vous avez utilisé la
syntaxe YAML dans votre fichier de configuration, puis suivez les
instructions à l'écran pour résoudre le problème.

Maintenant, arrêtez Prometheus en appuyant sur CTRL+C, puis ouvrez un
nouveau fichier de service systemd.
```
sudo nano /etc/systemd/system/prometheus.service
```
Le fichier de service `systemd` indique d'exécuter Prometheus en tant
qu'utilisateur **prometheus** , avec le fichier de configuration situé
dans le répertoire `/etc/prometheus/prometheus.yml` et de stocker ses
données dans le répertoire `/var/lib/prometheus`. ( vous pouvez en savoir plus sur [Présentation desunités Systemd et des fichiers d'unité](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files#where-are-systemd-unit-files-found)
.)

Recopiez le contenu suivant dans le fichier:

Fichier de service Prometheus - /etc/systemd/system/prometheus.service
```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```
Enfin, enregistrez le fichier et fermez votre éditeur de texte.

Pour utiliser le service nouvellement créé, rechargez systemd.
```
sudo systemctl daemon-reload
```
Vous pouvez maintenant démarrer Prometheus à l'aide de la commande
suivante:
```
sudo systemctl start prometheus
```
Pour vous assurer que Prometheus est en cours d'exécution, vérifiez
l'état du service.
```
sudo systemctl status prometheus
```
La sortie vous indique l'état de Prometheus, l'identifiant principal du
processus (PID), l'utilisation de la mémoire, etc.

Si l'état du service n'est pas active, suivez les instructions à l'écran
et retracez les étapes précédentes pour résoudre le problème avant de
poursuivre le cours.

Output
```
● prometheus.service - Prometheus
   Loaded: loaded (/etc/systemd/system/prometheus.service; disabled; vendor preset: enabl
   Active: active (running) since Tue 2021-02-09 18:50:33 UTC; 8s ago
 Main PID: 2209 (prometheus)
    Tasks: 9
   Memory: 19.8M
      CPU: 115ms
   CGroup: /system.slice/prometheus.service
           \u2514\u25002209 /usr/local/bin/prometheus --config.file /etc/prometheus/prometheus.yml
...
```
Lorsque vous êtes prêt à continuer, appuyez sur Q pour quitter la
commande status.

Enfin, activez le service au démarrage.
```
sudo systemctl enable prometheus
```
Maintenant que Prometheus est opérationnel, nous pouvons installer un
exportateur supplémentaire pour générer des métriques sur les ressources
de notre serveur.

## Étape 5 - Téléchargement de Node Exporter

Pour étendre Prometheus au-delà des métriques sur lui-même uniquement,
nous allons installer un exportateur supplémentaire appelé **Node Exporter**.
Node Exporter fournit des informations détaillées sur le système,
notamment l'utilisation du processeur, du disque et de la mémoire.

Tout d'abord, téléchargez la version stable actuelle de Node Exporter
dans votre répertoire personnel. Vous pouvez trouver les derniers
binaires ainsi que leurs sommes de contrôle sur [la page de
téléchargement de Prometheus](https://prometheus.io/download/) .
```
cd ~
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.1.0/node_exporter-1.1.0.linux-amd64.tar.gz
```
Utilisez la sha256sum commande pour générer une somme de contrôle du
fichier téléchargé:
```
sha256sum node_exporter-1.1.0.linux-amd64.tar.gz
```
Vérifiez l'intégrité du fichier téléchargé en comparant sa somme de
contrôle avec celle de la page de téléchargement.

Output
```
a894a85c58388b318843a586ca7540c1c772990ac5327e0c02e10db722f55a22
node_exporter-1.0.1.linux-amd64.tar.gz
```
Si les sommes de contrôle ne correspondent pas, supprimez le fichier
téléchargé et répétez les étapes précédentes.

Maintenant, décompressez l'archive téléchargée.
```
tar xvf node_exporter-1.1.0.linux-amd64.tar.gz
```
Cela créera un répertoire appelé node_exporter-1.1.0.linux-amd64
contenant un fichier binaire nommé node_exporter, une licence et un
avis.

Copiez le binaire dans le répertoire /usr/local/bin et définissez la
propriété de l'utilisateur et du groupe à l' utilisateur
**node_exporter** que vous avez créé à l'étape 1.
```
sudo cp node_exporter-1.1.0.linux-amd64/node_exporter /usr/local/bin
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```
Enfin, supprimez les fichiers restants de votre répertoire personnel car
ils ne sont plus nécessaires.
```
rm -rf node_exporter-1.1.0.linux-amd64.tar.gz node_exporter-1.0.1.linux-amd64
```
Maintenant que vous avez installé Node Exporter, testons-le en
l'exécutant avant de créer un fichier de service pour lui afin qu'il
se lance au démarrage.

## Étape 6 - Exécution de Node Exporter

Les étapes pour exécuter Node Exporter sont similaires à celles pour
exécuter Prometheus lui-même. Commencez par créer le fichier de service
Systemd pour Node Exporter.
```
sudo nano /etc/systemd/system/node_exporter.service
```
Ce fichier de service indique à votre système d'exécuter Node Exporter
en tant qu'utilisateur **node_exporter** avec l'ensemble de collecteurs
par défaut activé.

Copiez le contenu suivant dans le fichier de service:

Fichier de service Node Exporter - /etc/systemd/system/node_exporter.service
```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
Les collecteurs définissent les métriques que Node Exporter générera.
Vous pouvez voir la liste complète des collecteurs de Node Exporter - y
compris ceux qui sont activés par défaut et ceux qui sont obsolètes -
dans le [fichier README de Node Exporter](https://github.com/prometheus/node_exporter/blob/master/README.md#enabled-by-default).

Si vous devez remplacer la liste par défaut des collecteurs, vous pouvez
utiliser l'indicateur --collectors ( pour la liste des collecteurs [visitez](https://github.com/prometheus/node_exporter))

Enfin, rechargez systemd pour utiliser le service nouvellement créé.
```
sudo systemctl daemon-reload
```
Vous pouvez maintenant exécuter Node Exporter à l'aide de la commande
suivante:
```
sudo systemctl start node_exporter
```
Vérifiez que Node Exporter fonctionne correctement avec la commande
`status`.
```
sudo systemctl status node_exporter
```
Comme auparavant, cette sortie vous indique l'état de Node Exporter,
l'identificateur de processus principal (PID), l'utilisation de la
mémoire, etc.

Si l'état du service n'est pas active, suivez les messages à l'écran et
retracez les étapes précédentes pour résoudre le problème avant de
continuer.

Output
```
● node_exporter.service - Node Exporter
   Loaded: loaded (/etc/systemd/system/node_exporter.service; disabled; vendor preset: en
   Active: active (running) since Tue 2021-02-09 19:27:22 UTC; 6s ago
 Main PID: 2351 (node_exporter)
    Tasks: 5
   Memory: 2.5M
      CPU: 7ms
   CGroup: /system.slice/node_exporter.service
           \u2514\u25002351 /usr/local/bin/node_exporter

```
Enfin, activez Node Exporter pour démarrer au démarrage.
```
sudo systemctl enable node_exporter
```
Avec Node Exporter entièrement configuré et fonctionnant comme prévu,
nous dirons à Prometheus de commencer à récupérer les nouvelles métriques.

## Étape 7 - Configuration de Prometheus pour lire Node Exporter

Étant donné que Prometheus ne suit que les exportateurs définis dans
la partie scrape_configs de son fichier de configuration, nous devons
ajouter une entrée pour Node Exporter, tout comme nous l'avons fait pour
Prometheus lui-même.

Ouvrez le fichier de configuration.
```
sudo nano /etc/prometheus/prometheus.yml
```
À la fin du bloc `scrape_configs`, ajoutez une nouvelle entrée appelée
node_exporter.

Fichier de configuration Prometheus partie 1 - /etc/prometheus/prometheus.yml
```
...

- job_name: 'node_exporter'
  scrape_interval: 5s
  static_configs:
  - targets: ['localhost:9100']
```
Parce que cet exportateur est en cours d'exécution aussi sur le même
serveur que Prometheus, nous pouvons utiliser localhost au lieu d'une
adresse IP à nouveau avec le port par défaut du Node Exporter, 9100.

L'ensemble de votre fichier de configuration devrait ressembler à ceci:

Fichier de configuration Prometheus - /etc/prometheus/prometheus.yml
```
global:
  scrape_interval: 15s

scrape_configs:
- job_name: 'prometheus'
  scrape_interval: 5s
  static_configs:
  - targets: ['localhost:9090']
- job_name: 'node_exporter'
  scrape_interval: 5s
  static_configs:
  - targets: ['localhost:9100']
```
Enregistrez le fichier et quittez votre éditeur de texte lorsque vous
êtes prêt à continuer.

Enfin, redémarrez Prometheus pour appliquer les modifications.
```
sudo systemctl restart prometheus
```
Encore une fois, vérifiez que tout fonctionne correctement avec la
commande status.
```
sudo systemctl status prometheus
```
Si l'état du service n'est pas défini sur active, suivez les
instructions à l'écran et retracez vos étapes précédentes avant de
continuer.

Nous avons maintenant Prometheus et Node Exporter installés, configurés
et en cours d'exécution. Comme dernière précaution avant de se connecter
à l'interface Web, nous améliorerons la sécurité de notre installation
avec une authentification HTTP de base pour garantir que les
utilisateurs non autorisés ne peuvent pas accéder à nos métriques.

## Étape 8 - Sécurisation de Prometheus

Prometheus n'inclut pas d'authentification intégrée ni aucun autre
mécanisme de sécurité à usage général. D'une part, cela signifie que
vous obtenez un système très flexible avec moins de contraintes de
configuration; d'un autre côté, cela signifie que c'est à vous de vous
assurer que vos mesures et votre configuration globale sont suffisamment
sécurisées.

Par souci de simplicité, nous utiliserons Nginx pour ajouter une
authentification HTTP de base à notre installation, que Prometheus et
son outil de visualisation de données préféré, Grafana, prennent
entièrement en charge.

Commencez par installer apache2-utils, ce qui vous donnera accès à
l'utilitaire htpasswd de génération de fichiers de mots de passe.
```
sudo apt-get update
sudo apt-get install apache2-utils
```
Maintenant, créez un fichier de mots de passe en indiquant `htpasswd` où
vous souhaitez stocker le fichier et quel nom d'utilisateur vous
souhaitez utiliser pour l'authentification.

**Remarque:** htpasswd vous invite à saisir et à confirmer à nouveau le
mot de passe que vous souhaitez associer à cet utilisateur. Notez
également le nom d'utilisateur et le mot de passe que vous entrez ici,
car vous en aurez besoin pour vous connecter à Prometheus à l'étape 9.
```
sudo htpasswd -c /etc/nginx/.htpasswd sammy
```
Le résultat de cette commande est un fichier nouvellement créé appelé
`.htpasswd`, situé dans le repertoire `/etc/nginx`, contenant le nom
d'utilisateur et une version hachée du mot de passe que vous avez entré.

Ensuite, configurez Nginx pour utiliser les mots de passe nouvellement
créés.

Tout d'abord, faites une copie spécifique à Prometheus du fichier de
configuration par défaut de Nginx afin de pouvoir revenir aux valeurs
par défaut plus tard si vous rencontrez un problème.
```
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/prometheus
```
Ensuite, ouvrez le nouveau fichier de configuration.
```
sudo nano /etc/nginx/sites-available/prometheus
```
Localisez le bloc `location /` sous le bloc `server`. Cela devrait
ressembler à:

/etc/nginx/sites-available/default
```
...

location / {
  try_files $uri $uri/ =404;
}

...
```
Comme nous transmettrons tout le trafic à Prometheus, remplacez la
directive `try_files` par le contenu suivant:

/etc/nginx/sites-available/prometheus
```
...

location / {
  auth_basic "Prometheus server authentication";
  auth_basic_user_file /etc/nginx/.htpasswd;
  proxy_pass http://localhost:9090;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection 'upgrade';
  proxy_set_header Host $host;
  proxy_cache_bypass $http_upgrade;

}
...
```
Ces paramètres garantissent que les utilisateurs devront s'authentifier
au début de chaque nouvelle session. De plus, le proxy inverse dirigera
toutes les demandes traitées par ce bloc vers Prometheus.

Lorsque vous avez terminé vos modifications, enregistrez le fichier et
fermez votre éditeur de texte.

Désactivez maintenant le fichier de configuration par défaut de Nginx en
supprimant le lien vers celui-ci dans le répertoire
`/etc/nginx/sites-enabled` et activez le nouveau fichier de configuration
en créant un lien vers celui-ci.
```
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/prometheus /etc/nginx/sites-enabled/
```
Avant de redémarrer Nginx, vérifiez la configuration des erreurs à
l'aide de la commande suivante:
```
sudo nginx -t
```
La sortie doit indiquer que le syntax is ok et le test is successful. Si
vous recevez un message d'erreur, suivez les instructions à l'écran pour
résoudre le problème avant de passer à l'étape suivante.
```
Output of Nginx configuration tests
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Ensuite, rechargez Nginx pour incorporer toutes les modifications.
```
sudo systemctl reload nginx
```
Vérifiez que Nginx est opérationnel.
```
sudo systemctl status nginx
```
Si votre sortie n'indique pas que l'état du service est active, suivez
les messages à l'écran et retracez les étapes précédentes pour résoudre
le problème avant de continuer.
Output
```
● nginx.service - A high performance web server and a reverse proxy
server
Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor
preset: en
Active: active (running) since Mon 2017-07-31 21:20:57 UTC; 12min ago
Process: 4302 ExecReload=/usr/sbin/nginx -g daemon on; master_process
on; -s r
Main PID: 3053 (nginx)
Tasks: 2
Memory: 3.6M
CPU: 56ms
CGroup: /system.slice/nginx.service
```
À ce stade, nous avons un serveur Prometheus entièrement fonctionnel et
sécurisé, nous pouvons donc nous connecter à l'interface Web pour
commencer à examiner les métriques.

## Étape 9 - Test de Prometheus

Prometheus fournit une interface Web de base pour surveiller son statut
et celui de ses exportateurs, exécuter des requêtes et générer des
graphiques. Mais, en raison de la simplicité de l'interface, l'équipe
Prometheus [recommande d'installer et d'utiliser](https://prometheus.io/docs/visualization/browser/)  [Grafana](https://prometheus.io/docs/visualization/grafana/).

Dans ce cours, nous utiliserons l'interface Web intégrée pour nous
assurer que Prometheus et Node Exporter sont opérationnels, et nous
examinerons également des requêtes et des graphiques simples.

Pour commencer, pointez votre navigateur Web sur http://your_server_ip

Dans la boîte de dialogue d'authentification HTTP, entrez le nom
d'utilisateur et le mot de passe que vous avez choisis à l'étape 8.

Une fois connecté, vous verrez le **Expression Browser** , où vous
pouvez exécuter et visualiser des requêtes personnalisées.

Avant d'exécuter des expressions, vérifiez l'état de Prometheus et de
Node Explorer en cliquant d'abord sur le menu **État** en haut de
l'écran, puis sur l' option de menu **Cibles** . Comme nous avons
configuré Prometheus pour gratter à la fois lui-même et Node Exporter,
vous devriez voir les deux cibles répertoriées dans l'état **UP**.

Si l'un des exportateurs est manquant ou affiche un message d'erreur,
vérifiez l'état du service à l'aide des commandes suivantes:
```
sudo systemctl status prometheus
sudo systemctl status node_exporter
```
La sortie des deux services doit indiquer un état de *Active: active (running)*.
Si un service n'est pas actif du tout ou est actif mais ne fonctionne toujours pas
correctement, suivez les instructions à l'écran et retracez les étapes
précédentes avant de continuer.

Ensuite, pour nous assurer que les exportateurs fonctionnent
correctement, nous allons exécuter quelques expressions contre Node
Exporter.

Tout d'abord, cliquez sur le menu **Graphique** en haut de l'écran pour
revenir au **navigateur d'expression** .

Dans le champ **Expression** , saisissez `node_memory_MemAvailable_bytes` et
appuyez sur le bouton **Exécute** pour mettre à jour l' onglet **Console**
avec la quantité de mémoire dont dispose votre serveur.

Par défaut, Node Exporter signale cette valeur en octets. Pour convertir
en mégaoctets, nous utiliserons des opérateurs mathématiques pour
diviser par 1024 deux fois.

Dans le champ **Expression** , entrez `node_memory_MemAvailable_bytes/1024/1024`,
puis appuyez sur le bouton **Exécute**.

L'onglet **Table** affichera désormais les résultats en mégaoctets.

Si vous souhaitez vérifier les résultats, exécutez la commande `free`
depuis votre terminal. (Le drapeau `-h` indique à free de faire un rapport
dans un format lisible par l'homme, nous donnant le résultat en mégaoctets.)
```
free -h
```
Cette sortie contient des détails sur l'utilisation de la mémoire, y
compris la mémoire disponible affichée dans la colonne **disponible** .

Output
```
     total used free shared buff/cache available
Mem:  488M 144M 17M  3.7M   326M       324M
Swap: 0B 0B 0B
```
En plus des opérateurs de base, le langage de requête Prometheus fournit
également de nombreuses fonctions pour agréger les résultats.

Dans le champ **Expression** , saisissez
`avg_over_time(node_memory_MemAvailable_bytes[5m])/1024/1024` et cliquez
sur le bouton **Exécute**. Le résultat sera la mémoire moyenne
disponible au cours des 5 dernières minutes en mégaoctets.

Maintenant, cliquez sur l'onglet **Graph** pour afficher
l'expression exécutée sous forme de graphique plutôt que sous forme de
texte.

Enfin, tout en restant dans cet onglet, passez votre souris sur le
graphique pour plus de détails sur tout point spécifique le long des
axes X et Y du graphique.

Si vous souhaitez en savoir plus sur la création d'expressions dans
l'interface Web intégrée de [Prometheus](https://prometheus.io/docs/querying/basics/) , consultez la
partie [Interrogation de Prometheus](https://prometheus.io/docs/querying/basics/) de la
documentation officielle.

## Conclusion

Dans ce cours, nous avons téléchargé, configuré, sécurisé et testé
une installation complète de Prometheus avec un exportateur
supplémentaire.

Si vous souhaitez en savoir plus sur le fonctionnement de Prometheus
sous le capot, consultez [Comment interroger Prometheus sur Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-query-prometheus-on-ubuntu-14-04-part-1#step-2-%E2%80%94-installing-the-demo-instances). (Comme vous avez déjà installé Prometheus, vous pouvez ignorer la première étape.)

Pour voir ce que Prometheus peut faire d'autre, visitez
[la documentation officielle de Prometheus](https://prometheus.io/docs/introduction/overview/).

Et, pour en savoir plus sur l'extension de Prometheus, consultez la
[liste des exportateurs disponibles](https://prometheus.io/docs/instrumenting/exporters/) ainsi que le [site Web officiel de Grafana](https://grafana.com/) .
