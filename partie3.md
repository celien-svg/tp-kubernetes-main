## Partie 3

### CronJob
Un CronJob vous permet d'exécuter des tâches planifiées sur votre cluster Kubernetes. Cela peut être utile pour des tâches de maintenance, des rapports réguliers, des backups, etc.

1. 📄 Créer un fichier YAML `cronjob.yml`.
Dans ce fichier, ajoutez-y une ressource de type `CronJob` qui a pour nom `date`. Ce cronjob doit être exécuté toutes les 5 minutes et doit retourner le résultat de la commande `date`

---
*Livrables attendus : 1 fichier YAML, la liste des commandes utilisées*

### HPA

Un Horizontal Pod Autoscaling ajuste automatiquement le nombre de pods dans un déploiement en fonction de l'utilisation des ressources (CPU/mémoire).

1. 📄 Créer un fichier YAML `application.yml`. Dans ce fichier, ajoutez-y une ressource de type `Deployment` qui a pour nom `mon-application`. Vous pouvez utiliser l'image nginx ou n'importe quelle image du docker hub puis indiquer un `containerPort: 80`

2. Ajouter des requests/limits à cette application, le minimum recommandé par le propriétaire de l'image

3. 📄 Créer un fichier YAML `hpa-application.yml`. Dans ce fichier, ajoutez-y une ressource de type `HPA` qui a pour nom `mon-application`. L'HPA doit être cohérent avec les requests/limits pour pouvoir déclencher la création d'un pod supplémentaire lors de votre test local.

4. ❓ Avant de poursuivre, expliquez le résultat souhaité.

5. Exposer l'application localement sur le port 80
> kubectl expose deployment mon-application --port=80 --type=LoadBalancer

6. Vérifiez le HPA. Quelle commande avez-vous utilisé ? 

7. ❓ Expliquez les différentes colonnes retournées par la commande

8. 📸 Pour générer de la charge sur votre application, vous pouvez utiliser `kubectl run` pour créer une charge avec l'image `busybox` :

> kubectl run -i --tty simulation-utilisateur --image=busybox /bin/sh

Accédez au pod puis lancer la boucle suivante : 

> while true; do wget -q -O- http://mon-application.<namespace>.svc.cluster.local; done

9. ❓ Votre HPA fonctionne-t-il comme souhaité ? Ajustez le si nécessaire

---
*Livrables attendus : 2 fichier YAML, la liste des commandes utilisées et des captures d'écrans*

