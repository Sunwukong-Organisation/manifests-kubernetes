# manifests-kubernetes

Contexte
Le but de ce projet est de déployer wordpress et mysql dans un cluster kubernetes en suivant ces étapes :

Créez un déploiement MySQL avec une seule réplique,
Créez un clusterIP de type de service pour exposer les pods MySQL
Créez un déploiement WordPress avec toutes les variables d'environnement pour vous connecter à la base de données MySQL
Créez un port de nœud de type service pour exposer le frontend wordpress
Architecture
image

Étape 1 : Création de l'espace de noms
Pour séparer mes environnements des autres, j'ai créé un espace de noms wordpress:

apiVersion: v1
kind: Namespace
metadata: 
  name: wordpress
Étape 2 : Volumes persistants et réclamations de volumes persistants
Selon mon architecture, il existe deux volumes persistants (PV) et une réclamation de volumes persistants (pvc) pour chaque service. L'utilisation d'un volume persistant permet la persistance des données entre les modifications et le redémarrage du conteneur.

Le PV et le PVC sont configurés pour fonctionner comme ReadWriteOnce. MySQL PV a une capacité de 4Gi tandis que Wordpress PVC a une capacité de 3Gi.

#MySQL PV and PVC
apiVersion: v1 
kind: PersistentVolume
metadata:
  name: mysql-pv
  namespace: wordpress
spec:
  storageClassName: manual
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 4Gi
  hostPath:
    path: /mnt/mysql-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: wordpress
spec:
  resources:
    requests:
      storage: 3Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
#Wordpress PV and PVC
apiVersion: v1 
kind: PersistentVolume
metadata:
  name: wordpress-pv
  namespace: wordpress
spec:
  storageClassName: manual
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 2Gi
  hostPath:
    path: /mnt/mysql-wordpress
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: wordpress
spec:
  resources:
    requests:
      storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
Étape 3 : Secrets
Pour éviter l'exposition des mots de passe dans les données et améliorer la sécurité, j'ai utilisé l'objet Secretpour crypter les mots de passe en base64. Par exemple pour chiffrer une chaîne en base64, je le fais echo -n 'password' | openssl base64et cela me génère la chaîne chiffrée en base64.

apiVersion: v1
kind: Secret
metadata:
  name: wordpress-secret
  namespace: wordpress
type: Opaque
data:
  WORDPRESS_PASSWORD: d29yZHByZXNz
  MYSQL_ROOT_PASSWORD: bXlzcWw=
Étape 4 : Déploiements
Voici les déploiements que j'ai utilisés pour MySQL et WordPress. Les secrets ont été utilisés pour récupérer un mot de passe sensible et éviter l'exposition des données. Le type de stratégie recreate permet de s'assurer que les anciens pods sont toujours supprimés avant d'en générer de nouveaux.

#MySQL deployment
apiVersion: apps/v1
kind: Deployment 
metadata:
  name: wp-mysql
  namespace: wordpress
  labels:
    app: wordpress
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
      tier: backend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: backend
    spec:
      containers: 
      - name: mysql-container
        image: mysql
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wordpress-secret
              key: WORDPRESS_PASSWORD
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
             name: wordpress-secret
             key: MYSQL_ROOT_PASSWORD
        volumeMounts:
         - name: mysql-data
           mountPath: /var/lib/mysql
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-pvc
#Wordpress deployment
apiVersion: apps/v1
kind: Deployment 
metadata:
  name: wordpress
  namespace: wordpress
  labels:
    app: wordpress
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers: 
      - name: wordpress-container
        image: wordpress:latest
        ports:
        - containerPort: 80
        env:
        - name: WORDPRESS_DB_USER
          value: wordpress
        - name: WORDPRESS_DB_NAME
          value: wordpress
        - name: WORDPRESS_DB_HOST
          value: wp-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wordpress-secret
              key: WORDPRESS_PASSWORD
        volumeMounts:
        - name: wordpress-data
          mountPath: /var/www/html
      volumes:
      - name: wordpress-data
        persistentVolumeClaim:
          claimName: wordpress-pvc
Étape 5 : Prestations
J'ai créé un service clusterIP (backend) pour exposer l'application MySQL à l'intérieur du cluster Kubernetes et un service nodePort (frontend) pour accéder à l'application en dehors du cluster.

#MySQL clusterIP Service
apiVersion: v1 
kind: Service
metadata:
  name: wp-mysql
  namespace: wordpress
spec:
  selector:
    app: wordpress
    tier: backend
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
#Wordpress nodePort Service
apiVersion: v1 
kind: Service
metadata:
  name: wordpress-svc
  namespace: wordpress
spec:
  selector:
    app: wordpress
    tier: frontend
  type: NodePort
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 80
    nodePort: 30080
Étape 6 : Entrée
Pour accéder à l'application et la livraison à un client, nous ne pouvons pas la laisser accessible via IP virtuelle : Port . Pour livrer une bonne application au client, j'ai utilisé Ingressun objet pour exposer l'application wordpress avec un nom de domaine personnalisé.

Les paramètres suivants me permettent de transférer l'application vers le nom de domaine http://dev-wordpress.fr .

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-ingress
  namespace: wordpress
spec:
  rules:
  - host: dev.wordpress.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wordpress-svc
            port:
              number: 8080
Étape 7 : Déployer tout en un
Pour automatiser le déploiement de l'application dans n'importe quelle infrastructure, nous pouvons soit copier-coller tous les manifestes dans un fichier YML et le lancer avec :

kubectl create -f deploy-wordpress-app.yml

Ou placez tous les manifestes dans un seul dossier et lancez-le avec :

kubectl apply -f ./

Les deux méthodes sont utiles car vous pouvez toujours supprimer tous les déploiements en une seule commande :

kubectl delete -f ./oukubectl delete -f deploy-wordpress-app.yml

Étape 8 : Profiter de l'application
Après avoir déployé l'application en une seule commande avec la précédente, vous pouvez accéder à Wordpress en utilisant l'adresse http://dev-wordpress.fr liée à une base de données mysql.

Conclusion
