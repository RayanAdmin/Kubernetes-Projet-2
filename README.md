# Kubernetes-Projet-2

# Introduction

Le but de ce projet est de déployer une application PHP qui affiche le contenu d'une base de données MySQL sur Kubernetes en utilisant un serveur NFS pour stocker les données de la base de données et les rendrent ainsi persistantes.

À la différence du projet numéro 1, un répertoire est partagé entre le pod MySQL et le répertoire sur le serveur NFS. Ainsi, si le pod MySQL est supprimé et que le pod est redeployé, la base de données n'est pas supprimée car les données sont stockées sur le serveur NFS. De plus, fait d'utiliser un serveur NFS et y monter le répertoire partagé permet d'avoir accès au contenu de ce répertoire depuis n'importe quel machine sur le reseau.

## Objectif

L'objectif de ce projet est de montrer comment utiliser Kubernetes pour déployer une application web en utilisant des pods et des services, ainsi que pour rendre les données de la base de données persistantes en utilisant un serveur NFS.

## Installation et Configuration du serveur NFS

1. Installer le serveur NFS sur la machine où Kubernetes est installé en utilisant la commande suivante :

```
sudo apt-get install nfs-kernel-server
```

2. Créer un répertoire partagé en exécutant la commande suivante :

```
sudo mkdir /home/rayan/projet_02/nfs
```

Note : Remplacez `/home/rayan/projet_02/nfs` par le repertoire qui va être monté sur le pod.

3. Modifier les permissions du répertoire partagé en exécutant la commande suivante :

```
sudo chmod 777 /home/rayan/projet_02/nfs
```

Note : Remplacez `/home/rayan/projet_02/nfs` par le repertoire qui va être monté sur le pod.

4. Ajouter les paramètres de partage pour le répertoire partagé en modifiant le fichier `/etc/exports` avec les informations suivantes :

```
/home/rayan/projet_02/nfs *(rw,sync,no_subtree_check)
```

Note : Remplacez `/home/rayan/projet_02/nfs` par le repertoire qui va être monté sur le pod.

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
  storageClassName: my-nfs-storage
  accessModes:
    - ReadWriteMany
  nfs:
    path: /home/rayan/projet_02/nfs
    server: <IP-address-of-Kubernetes-master-node>
```
Note : Remplacez `/home/rayan/projet_02/nfs` par le repertoire qui va être monté sur le pod.
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
  storageClassName: my-nfs-storage
  resources:
    requests:
      storage: 1Gi
  volumeName: my-nfs-pv
```

## Configuration des Pods et Services

1. Créer un pod MySQL en utilisant le fichier suivant :

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mydb-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      env: production-db
  template:
    metadata:
      name: mydb
      labels:
        env: production-db
    spec:
      securityContext:
        runAsUser: 1001
        fsGroup: 1002
      volumes:
        - name: my-nfs-storage
          persistentVolumeClaim:
            claimName: my-nfs-pvc
      containers:
      - name: database
        image: mysql:5.7
        ports:
        - containerPort: 3306
        volumeMounts:
          - name: my-nfs-storage
            mountPath: /var/lib/mysql
        env:
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-pass
                key: password
          - name: MYSQL_DATABASE
            value: db
```


2. Créer un pod PHP en utilisant le fichier suivant :

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myphp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      env: production-frontend
  template:
    metadata:
      name: myfrontend-pod
      labels:
        env: production-frontend
    spec:
      volumes:
        - name: php-files
          hostPath:
            path: /home/rayan/projet_01/site
      containers:
        - name: frontend
          image: ragh19/phpproject:web_v1
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: password
          ports:
            - containerPort: 80
          volumeMounts:
            - mountPath: /var/www/html
              name: php-files
```

3. Créer le service Mysql en utilisant le fichier suivant :

```
apiVersion: v1
kind: Service
metadata:
  name: mydb-service
spec:
  type: ClusterIP
  ports:
    - targetPort: 3306
      port: 3306
#     nodePort: 30008 
  selector:
    env: production-db
``` 

4. Créer le service PHP en utilisant le fichier suivant :

```
apiVersion: v1
kind: Service
metadata:
  name: myphp-service
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30009
  selector:
    env: production-frontend
``` 

## Configuration du fichier de Kustomization

1. Créer le fichier de Kustomization en utilisant le fichier suivant :

```
secretGenerator:
- name: mysql-pass
  literals:
  - password=redhat
resources:
  - 02-deployment-db.yaml
  - 02-deployment-php.yaml
``` 
