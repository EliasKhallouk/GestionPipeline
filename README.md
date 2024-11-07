<img src="logo-uqac.png" alt="logo UQAC" align="right">

# 8CLD201 - Devoir 1<!-- omit from toc -->
> Antoine Crauser - Breval Ferrari - Elias Khallouk

Ce projet correspond au rendu du devoir 1 du cours "8CLD201 - Environnement de déploiement des applications".

Les consignes peuvent être retrouvées dans [consignes.md](./consignes.md).

Ce projet est sur [Azure DevOps](https://dev.azure.com/eliaskhallouk/GestionPipeline) et sur [GitHub](https://github.com/EliasKhallouk/GestionPipeline).

- [Annexes](#annexes)
  - [`Cloud-init.txt`](#cloud-inittxt)


## Annexes
### `Cloud-init.txt`
Voici le fichier [Cloud-init.txt](./AzureRessourceGroup1/Cloud-init.txt) expliqué pas à pas :
```yml
package_upgrade: true
```
Nous autorisons d'abord Azure à mettre à jour les paquets utilisés par ce script. En voici la liste :
```yml
packages:
  - nginx
  - nodejs
  - npm
```
Ces paquets sont `nginx`, un serveur HTTP, `nodejs`, un interpréteur de Node.js qui fournit une *runtime* JavaScript, et `npm`, un gestionnaire de projet.

Nous avons ensuite une section où il est possible de créer des fichiers sur la machine virtuelle :
```yml
write_files:
  - owner: www-data:www-data
```
Le premier est le fichier de configuration du site par défaut déployé par `nginx`, qui appartient à l'utilisateur virtuel `www-data` et au groupe d'accès `www-data`, ce qui permet au serveur HTTP et donc au client d'avoir accès à ces fichiers.
```yml
    path: /etc/nginx/sites-available/default
    defer: true
```
Voici son contenu :
```yml
    content: |
      server {
        listen 80;
        location / {
          proxy_pass http://localhost:3000;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection keep-alive;
          proxy_set_header Host $host;
          proxy_cache_bypass $http_upgrade;
        }
      }
```
Cette configuration commande entre autres une écoute du serveur sur le port 80 (HTTP) et un proxy sur le port 3000.

Le dernier fichier est le site même, un script qui renvoie un "Hello World" aux clients connectés sur le port 3000 :
```yml
  - owner: www-data:www-data
    path: /home/myapp/index.js
    defer: true
    content: |
      var express = require('express')
      var app = express()
      var os = require('os');
      app.get('/', function (req, res) {
        res.send('Hello World from host ' + os.hostname() + '!')
      })
      app.listen(3000, function () {
        console.log('Hello world app listening on port 3000!')
      })
```
Ce script affiche aussi sur appel de la racine du document un message contenant le nom d'hôte de la machine sur laquelle il tourne. Cela nous permet de tester notre *load-balancer*.

La dernière partie du `Cloud-init.txt` est une liste de commandes à exécuter par Azure pour faire tourner notre serveur :
```yml
runcmd:
  - service nginx restart
  - cd "/home/myapp"
  - npm init
  - npm install express -y
  - nodejs index.js
```
Voici un résumé des étapes du script :
1. redémarrer le serveur HTTP `nginx`
1. naviguer dans le dossier source du site
1. télécharger les dépendances et préparer le site
1. installer `express`, un framework web Node.js
1. lancer le script vu précédemment