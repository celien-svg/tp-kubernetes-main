# TP Avancé : Rendre une Application "Production-Ready" sur Kubernetes

## Introduction

Dans ce TP, nous allons dépasser le simple déploiement pour aborder les problématiques d'un développeur qui doit garantir que son application est robuste, résiliente et performante en production.

Une équipe de développement vous a fourni une image Docker d'un microservice : `paulbouwer/hello-kubernetes:1.10`. Cette application est simple : elle affiche un message "Hello", quelques variables d'environnement et le nom du pod sur lequel elle s'exécute.

Votre mission est de prendre cette application "naïve" et de la transformer en un service de qualité "production" en utilisant uniquement les fonctionnalités de Kubernetes. **Vous ne modifierez pas le code de l'application, seulement la manière dont elle est déployée.**

## Livrables

- Un document (PDF, Word, Markdown...) avec vos réponses aux questions ❓, vos captures d'écran 📸, et vos réflexions.
- L'ensemble des fichiers YAML 📄 que vous avez créés ou modifiés.

## Pré-requis

- Un cluster Minikube fonctionnel.
- Les TP précédents terminés.

---

## Partie 1 : Déploiement

Un déploiement commence par des fondations saines. Une application qui consomme trop de ressources ou qui n'en a pas assez est une source de problèmes pour tout le cluster.

1.  **Déploiement Initial**
    - [ ] Créez un nouveau namespace `prod-app`.
    - [ ] 📄 Créez un fichier `app-deployment.yml` et `app-service.yml`.
    - [ ] Dans `app-deployment.yml`, définissez un `Deployment` qui déploie l'image `paulbouwer/hello-kubernetes:1.10` avec `3` réplicas.
    - [ ] Dans `app-service.yml`, définissez un `Service` de type `ClusterIP` pour exposer votre déploiement.
    - [ ] Déployez vos ressources dans le namespace `prod-app`.
    - [ ] Utilisez `kubectl port-forward` pour accéder à votre service depuis votre navigateur.
        > `kubectl port-forward -n prod-app svc/<nom-du-service> 8080:8080`
    - [ ] 📸 Accédez à `http://localhost:8080` sur votre navigateur. Rafraîchissez la page plusieurs fois.
    - [ ] ❓ Que remarquez-vous sur la valeur "Pod Name" affichée à chaque rafraîchissement ? Pourquoi cela se produit-il ?

2.  **Définir les `requests` et `limits` (Qualité de Service)**
    Actuellement, votre pod peut essayer de consommer toutes les ressources du nœud, ce qui est dangereux pour la stabilité du cluster. Nous allons lui imposer des limites.

    - [ ] Modifiez votre fichier `app-deployment.yml` pour ajouter une section `resources` à la définition du conteneur.
        ```yaml
        resources:
          requests:
            cpu: "100m"    # 10% d'un vCPU
            memory: "64Mi" # 64 Mebibytes
          limits:
            cpu: "200m"
            memory: "128Mi"
        ```
    - [ ] Appliquez les changements avec `kubectl apply -f app-deployment.yml`.
    - [ ] ❓ Kubernetes définit 3 classes de Qualité de Service (QoS) : `Guaranteed`, `Burstable`, `BestEffort`. En vous basant sur la configuration que vous venez de mettre en place, à quelle classe appartient votre pod ? Pourquoi est-il crucial pour un développeur de viser au minimum la classe `Burstable` pour une application de production ?

---

## Partie 2 : Probes

Votre application tourne, mais est-elle réellement en bonne santé ? Kubernetes a besoin de le savoir pour ne pas envoyer de trafic à un pod qui a planté ou qui n'est pas encore prêt.

*L'application `hello-kubernetes` a été bien conçue ! Elle expose les routes `/healthz` (pour la santé) et `/ready` (pour la disponibilité) sur son port `8080`.*

1.  **La `livenessProbe` : "Es-tu en vie ?"**
    - [ ] Modifiez votre `app-deployment.yml` pour ajouter une `livenessProbe` à la définition de votre conteneur. Elle doit faire une requête HTTP GET sur le chemin `/healthz`. [Documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-http-request)
    - [ ] Appliquez la configuration.
    - [ ] ❓ Que se passerait-il si le processus de l'application se bloquait (deadlock) sans pour autant crasher ? Comment la `livenessProbe` aiderait-elle à résoudre le problème automatiquement ?

2.  **La `readinessProbe` : "Es-tu prêt à recevoir du trafic ?"**
    - [ ] Ajoutez maintenant une `readinessProbe` qui utilise la route `/ready`.
    - [ ] Pour bien visualiser son effet, simulez une application qui met du temps à démarrer en ajoutant le paramètre `initialDelaySeconds: 10` à votre `readinessProbe`.
    - [ ] Appliquez les changements. Utilisez `kubectl describe pod <nom-du-pod> -n prod-app` pour observer les événements liés à la sonde.
    - [ ] 📸 Fournissez une capture d'écran de la section "Events" où l'on voit que la sonde attend avant de réussir.
    - [ ] ❓ Expliquez la différence de conséquence entre un échec de la `livenessProbe` et un échec de la `readinessProbe`. Quel composant de Kubernetes cesse d'envoyer du trafic au pod dans le second cas ?

---

## Partie 3 : Déploiements sans interruption et arrêt propre

Mettre à jour une application est une opération courante, mais elle ne doit jamais causer d'erreurs `503` pour vos utilisateurs.

1.  **Le `RollingUpdate`**
    - [ ] La stratégie de mise à jour par défaut d'un `Deployment` est `RollingUpdate`. Nous allons l'observer en action.
    - [ ] Modifiez votre `app-deployment.yml` pour utiliser une nouvelle version de l'image : `paulbouwer/hello-kubernetes:1.10.1`.
    - [ ] Appliquez le changement. Dans un autre terminal, lancez immédiatement la commande suivante pour observer le déploiement en direct :
        > `kubectl get pods -n prod-app -w`
    - [ ] 📸 Observez comment Kubernetes remplace les anciens pods par les nouveaux, un par un. Faites une capture d'écran.
    - [ ] ❓ Comment les `readinessProbes` que vous avez configurées à l'étape précédente sont-elles essentielles pour garantir un `RollingUpdate` sans aucune interruption de service ?

2.  **L'Arrêt "gracieux" (`graceful shutdown`)**
    Quand Kubernetes arrête un pod, il lui envoie un signal `SIGTERM`. Si l'application ne le gère pas, elle peut couper brutalement des requêtes en cours de traitement.

    - [ ] Ajoutez un `preStop` lifecycle hook à votre conteneur. Ce hook simulera le temps nécessaire à l'application pour finir son travail en cours.
        ```yaml
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
        ```
    - [ ] Appliquez ce changement.
    - [ ] Supprimez manuellement l'un des pods : `kubectl delete pod <nom-du-pod> -n prod-app`.
    - [ ] Observez rapidement son statut avec `kubectl get pod <nom-du-pod> -n prod-app`. Il devrait rester en statut `Terminating` pendant environ 10 secondes.
    - [ ] ❓ À quoi sert ce `preStop` hook ? Dans un vrai scénario applicatif, que ferait une application pendant ce temps de "grâce" au lieu de simplement dormir ?

---

## Partie 4 : Débogage avec des conteneurs éphémères

Parfois, on a besoin de déboguer un pod en production. L'image de base de notre application est minimale et n'a pas d'outils comme `curl` ou `ping`. Heureusement, Kubernetes a une solution propre.

1.  **Le problème :** Essayez de vous connecter à l'un de vos pods avec `kubectl exec` et de lancer la commande `curl localhost:8080`. Cette commande échouera car l'utilitaire `curl` n'est pas inclus dans l'image de l'application.
    
2.  **La solution `kubectl debug` :**
    - [ ] Ciblez un de vos pods en cours d'exécution et lancez un "conteneur de débogage" à côté de lui. Ce conteneur partagera le réseau de votre pod applicatif.
        > `kubectl debug -it <nom-de-votre-pod> -n prod-app --image=busybox --target=hello-kubernetes`
        *(Note : L'argument `--target=hello-kubernetes` est nécessaire pour indiquer à quel conteneur du pod on souhaite s'attacher, car un pod peut en avoir plusieurs).*
    - [ ] Vous devriez obtenir un shell dans ce nouveau conteneur de débogage. Depuis ce shell, lancez `wget -O - localhost:8080` (busybox utilise `wget` au lieu de `curl`).
    - [ ] 📸 Faites une capture d'écran du bon fonctionnement
    - [ ] ❓ Pourquoi cette approche est-elle considérée comme une bien meilleure pratique de sécurité et d'exploitation que d'inclure tous les outils de débogage (`curl`, `ping`, `vim`, etc.) directement dans l'image de votre application de production ?