# ollamazure

## Installation

Une fois la VM Azure créée (je vous passe les détails car la création de VM est déjà guidée en interface), nous pouvons démarrer l'installation de nos outils. On se connecte à la machine (via Putty ou Azure CLI), et on démarre l'installation d'Ollama. J'inclus également ci-dessous l'installation de l'outil htop, qui sert à voir l'utilisation des ressources de votre machine. Vous n'êtes pas obligé de l'installer si vous n'en voulez pas.
```bash
sudo apt update && sudo apt upgrade
curl -fsSL https://ollama.com/install.sh | sh
sudo apt install htop
```

On télécharge maintenant le modèle de notre choix, les noms de modèles pouvant être installés sont trouvables directement sur ollama.com. J'installe de mon côté une version légère de llama3.2.
```bash
ollama pull llama3.2:1b
```

Il nous reste une dernière chose à faire sur Ollama, faire en sorte qu'il écoute sur toutes les interfaces. Si vous ne faites pas cela Ollama ne pourra pas communiquer avec votre interface web tout-à-l'heure et vous n'aurez pas accès aux modèles que vous avez installé.
```bash
nano /etc/systemd/system/ollama/service
```

```bash
# Dans le bloc ci-dessous ajoutez cette variable d'environnement :
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
```

Il reste ensuite à redémarrer Ollama.
```bash
systemctl daemon-reload
systemctl restart ollama.service
```

✅ Ollama est installé et prêt à être utilisé. 

Passons à l'installation d'un outil de conteneurisation. Il est possible d'installer Open WebUI via Python, et Nginx via la commande apt, mais la conteneurisation présente plusieurs avantages. Elle nous permet de garantir une isolation des applications, réduisant alors le risque de conflits entre les dépendances, et elle facilite les mises à jour des différents services.

Je choisis d'installer Podman, une alternative à Docker développée par RedHat qu'il est possible d'installer en une seule commande. Je vais ensuite aller chercher les images d'Open WebUI et de Nginx.
```bash
sudo apt install -y podman
podman network create mynet
podman pull ghcr.io/open-webui/open-webui:main
podman pull docker.io/nginx
```

Pour que notre conteneur Open WebUI puisse communiquer avec notre hôte et donc interagir avec Ollama, nous devons obtenir l'adresse IP interne de notre hôte avec cette commande :
```bash
ip -4 addr show dev eth0
```

Nous pouvons ensuite déployer notre conteneur Open WebUI (changez l'IP via l'IP interne de votre hôte).
```bash
podman run -d --network mynet -p 3000:8080 --add-host=host.containers.internal:10.0.0.4 -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

✅ L'interface web Open WebUI est déployée.

Cependant elle n'est pas encore accessible, vous pouvez y accéder tout de suite en ouvrant le port 3000 de votre machine, mais je ne vous le recommande pas, vos communications n'étant pas encore chiffrées grâce au protocole HTTPS.

Pour le certificat j'utiliserais un certificat auto-signé mais vous pouvez utiliser un certificat Let's Encrypt si vous le souhaitez. Je commence donc par créer le dossier où seront stockées les clés publiques et privées. Dans la 2e commande n'oubliez pas de remplacer le CN (Common Name) par votre IP ou votre nom de domaine :
```bash
mkdir -p ~/nginx/cert

openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ~/nginx/cert/selfsigned.key -out ~/nginx/cert/selfsigned.crt -subj "/C=FR/ST=France/L=Paris/O=Company/CN=IP-ou-nomdedomaine"
```

Nous allons ensuite créer la configuration de notre reverse-proxy Nginx. Je limite le nombre de connexions simultanées à 1024. Vous remarquerez aussi l'ajout du support des WebSocket. Sans cela, lorsque l'on tente une requête vers un modèle d'IA on obtient une erreur car la génération du message de l'IA se fait en temps réel et nécessite ce type de connexion.
```bash
nano ~/nginx/nginx.conf
```

```conf
user nginx;
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    server {
        listen 80;
        listen 443 ssl;
        server_name IP-ou-nomdedomaine;  # Remplacez par votre IP/Nom de domaine

        ssl_certificate /etc/nginx/cert/selfsigned.crt;
        ssl_certificate_key /etc/nginx/cert/selfsigned.key;

        location / {
            proxy_pass http://open-webui:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Support WebSocket
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
        }
    }
}
```

On peut maintenant déployer notre conteneur Nginx, en prenant en compte l'emplacement de nos fichiers de configuration.
```bash
podman run -d --network mynet --name nginx -p 443:443 -v ~/nginx/nginx.conf:/etc/nginx/nginx.conf:ro -v ~/nginx/cert:/etc/nginx/cert:ro nginx
```

✅ Nginx est déployé et l'interface web Open WebUI est accessible.

## Accès à l'interface

Une fois ceci fait vous devriez pouvoir accéder à l'interface via le HTTPS. Si ce n'est pas le cas vérifiez que vous avez ouvert le port via le pare-feu Azure, vous devez avoir une règle qui ressemble à la 2e de l'image ci-dessous, sinon créez-là.

![Alt text](images/azure.png?raw=true "azure")

Sur l'interface on vous demande de créer votre premier utilisateur qui sera aussi Administrateur, et de lui attribuer un mot de passe, une fois ceci fait vous accédez à l'interface.

![Alt text](images/interface.png?raw=true "interfaces")

Vous n'accéderez pas tout de suite aux modèles car il reste une chose à changer. Par défaut Open WebUI utilise Docker mais nous utilisons Podman et cela pose un petit soucis dans les paramètres. Cliquez sur le logo de votre utilisateur -> Paramètres -> Paramètres admin. -> Connexions. Vous devez changer "http://host.docker.internal:11434" par ceci :

![Alt text](images/connection.png?raw=true "connection")

✅ Vous pouvez désormais accéder à vos modèles et commencer à discuter.

![Alt text](images/message.png?raw=true "presentation")

> N'oubliez pas d'explorer les options d'utilisateurs et de groupes d'utilisateurs afin de créer des utilisateurs pouvant uniquement communiquer avec les modèles que vous leur proposez. En session administrateur vous pouvez créer des modèles pour les utilisateurs, des prompts, ajouter une base de connaissances, et plus encore, ce qui fait beaucoup plus d'autorisations que nécessaire. Pensez à vous créer un utilisateur avec des droits appropriés.

## Conclusion

Vous savez maintenant comment installer Ollama sur une machine virtuelle déployée dans le Cloud, et communiquer avec les modèles proposés à travers une interface web open-source, le tout sécurisé via le protocole HTTPS. Bravo! 🎉