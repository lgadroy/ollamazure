# ollamazure

## Avant de commencer

Ce court guide vous expliquera pas à pas comment mettre en place un chatbot sur une machine virtuelle sur Azure. L'objectif est que ce chatbot soit accessible depuis notre navigateur en HTTPS et nous fasse nous connecter à un compte utilisateur pour accéder à l'interface.

Nous utiliserons donc Ollama, un outil open-source capable d'exécuter des modèles d'IA sur une machine 'locale'. L'outil est compatible avec Windows, Linux et MacOS. Ollama ne proposant pas d'interface web nativement, nous allons installer Open WebUI, une interface open-source compatible avec Ollama. Enfin nous configurerons Nginx pour rendre le tout accessible en HTTPS.

## Configuration

Ma machine est une VM Standard_B2ms, possédant 2 vCPU et 8 Gio de mémoire vive. C'est une machine à 'moindre' coût, mais c'est surtout le minimum pour faire tourner un modèle d'IA. Je ne pourrais faire tourner qu'un petit modèle avec 1 milliard de paramètres sur cette machine. Je vous recommande donc de vous renseigner sur le modèle que vous voulez utiliser en vous rendant sur le site d'Ollama avant de créer votre VM, afin de ne pas vous rendre compte trop tard que votre machine n'est pas assez puissante.

Pour l'OS j'ai choisi Debian sans interface graphique, car je connais assez bien cet OS et qu'il consomme beaucoup moins de ressources qu'un Windows. Je vous conseille également une distribution Linux sans interface graphique étant donné que vous n'aurez que des commandes à exécuter.

## Installation

Pour voir le tutoriel d'installation complet, [cliquez ici](install.md)

## Démonstration

Une fois le tutoriel terminé et un utilisateur créé, vous pouvez interagir avec le modèle de votre choix.
![Alt text](images/message.png?raw=true "presentation")

## Sources

- https://ollama.com/
- https://github.com/open-webui/open-webui
- https://github.com/open-webui/open-webui/discussions/9534
- https://nginx.org/en/docs/http/websocket.html
- https://www.youtube.com/watch?v=T8ym4SkRic0