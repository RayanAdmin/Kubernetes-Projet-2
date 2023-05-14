# Kubernetes-Projet-2

# Installation et Configuration d'un serveur NFS

## Introduction

Le but de ce projet est de déployer une application PHP qui affiche le contenu d'une base de données MySQL sur Kubernetes en utilisant un serveur NFS pour stocker les données de la base de données et les rendrent ainsi persistantes.

La différence avec le projet numéro 1 est la persistances des données sur le serveur MYSQL, ce qui permet que le contenue de la base de donnée ne soit pas perdu si le pod tombe ou se fait supprimer. De plus, le serveur NFS permet d'avoir accès au contenu de ce répertoire depuis n'importe quel machine sur le reseau.

## Objectif

L'objectif de ce projet est de créer une page web en PHP qui affiche le contenu d'une base de données MySQL. Pour cela, j'ai utilisé Kubernetes pour déployer un pod PHP et un pod MySQL, ainsi que deux services associés pour faciliter l'accès à l'application. Le fichier `kustomization.yaml` est utilisé pour partager le mot de passe de la base de données entre les deux pods.

À la différence du projet numéro 1, un répertoire est partagé entre le pod MySQL et le répertoire sur le serveur NFS. Ainsi, si le pod MySQL est supprimé et que le pod est redeployé, la base de données n'est pas supprimée car les données sont stockées sur le serveur NFS.

Puis, j'ai créé un déploiement pour chaque pod, ainsi qu'un service pour chaque déploiement. Le service du pod MySQL a été configuré pour avoir un accès en lecture-écriture, et le service du pod PHP a été configuré pour avoir un accès en lecture seule. 

J'ai également créé un fichier `kustomization.yaml`, qui est utilisé pour partager le mot de passe de la base de données entre les deux pods. Ce fichier permet également de référencer les fichiers YAML nécessaires pour déployer les différents objets Kubernetes.

Finalement, j'ai créé un fichier PHP pour afficher le contenu de la base de données. Ce fichier PHP est monté en tant que volume dans le pod PHP et utilise le service MySQL pour se connecter à la base de données.

Le déploiement de l'application peut être effectué en exécutant la commande `kubectl apply -k .` dans le répertoire racine du projet. Cela déploiera les objets Kubernetes nécessaires pour exécuter l'application, y compris les pods, les services et les volumes persistants.

L'objectif de ce projet est de montrer comment utiliser Kubernetes pour déployer une application web en utilisant des pods et des services, ainsi que pour rendre les données de la base de données persistantes en utilisant un serveur NFS.

## Installation et Configuration du serveur NFS

1. Installer le serveur NFS sur la machine où Kubernetes est installé en utilisant la commande suivante :

```
sudo apt-get install nfs-kernel-server
```

2. Créer un répertoire partagé en exécutant la commande suivante :

```
sudo mkdir ~/projet_02/nfs
```

3. Modifier les permissions du répertoire partagé en exécutant la commande suivante :

```
sudo chmod 777 ~/projet_02/nfs
```

4. Ajouter les paramètres de partage pour le répertoire partagé en modifiant le fichier `/etc/exports` avec les informations suivantes :

```
/home/<user>/projet_02/nfs *(rw,sync,no_subtree_check)
```

Note : Remplacez `<user>` par votre nom d'utilisateur.

5. Appliquer le fichier en utilisant la commande suivante :

```
sudo exportfs -rav
```

6. Redémarrer le serveur NFS en utilisant la commande suivante :

```
sudo systemctl restart nfs-kernel-server
```

## Configuration des Persistent Volumes et Persistent Volume Claims

1. Créer un Persistent Volume (PV) pour le serveur NFS en utilisant le fichier suivant :

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-nfs-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: ~/projet_02/nfs
    server: <IP-address-of-Kubernetes-master-node>
```

Note : Remplacez `<IP-address-of-Kubernetes-master-node>` par l'adresse IP du noeud maître Kubernetes.

2. Créer un Persistent Volume Claim (PVC) pour le serveur NFS en utilisant le fichier suivant :

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      type: nfs
```

## Configuration des Pods et Services

1. Créer un pod MySQL en utilisant le fichier suivant :

```
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  labels:
    type: nfs
spec:
  containers:
  - name: mysql-container
    image: mysql
    env:
      - name: MYSQL_ROOT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: password
    volumeMounts:
      - mountPath: /var/lib/mysql
        name: mysql-persistent-storage
  volumes:
  - name: mysql-persistent-storage
    persistentVolumeClaim:
      claimName: my-nfs-pvc
```


2. Créer un pod PHP en utilisant le fichier suivant :

```
apiVersion: v1
kind: Pod
metadata:
  name: php-pod
  labels:
    type: nfs
spec:
  containers:
  - name: php-container
    image: php:apache
    ports:
      - containerPort: 80
    volumeMounts:
      - mountPath: /var/www/html
        name: php-files
```
