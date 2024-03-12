# Déploiement de l'application front-end et back-end sur Minikube

Ce Readme décrit les étapes pour déployer une application front-end et back-end sur Minikube et la publication des images Docker .

# Prérequis
- Minikube
- Docker
- Istio

# Étapes de déploiement

## Démarrage de Minikube : 

Pour démarrer votre cluster Minikube, exécutez la commande suivante dans votre terminal :

```
minikube start
```

## Construction et publication des images Docker :

Etre dans le répertoire contenant vos applications front-end et back-end.

## Compilation du projet : 

Compilation du projet avec la commande suivante : 

```
./gradlew build
```

### Back-end : 

Création de l'image docker
Publication dans notre Docker Hub
```
docker build -t mb923/back-end:v1 .
docker tag mb923/back-end:v1 mb923/back-end:v1
docker login
docker push mb923/back-end:v1
```

### Front-end : 

Création de l'image docker
Publication dans notre Docker Hub
```
docker build -t mb923/front-end:v1 .
docker tag mb923/front-end:v1 mb923/front-end:v1
docker login
docker push mb923/front-end:v1
```

## Création du fichier YAML pour Kubernetes :

Création d'un fichier YAML nommé front-back-app.yml décrivant les déploiements et les services pour vos applications front-end et back-end. 

### Création du déploiement Kubernete

(petit texte) 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: front-end-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: front-end
  template:
    metadata:
      labels:
        app: front-end
    spec:
      containers:
      - name: front-end-container
        image: mb923/front-end:v1
        imagePullPolicy: Always
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: back-end-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: back-end
  template:
    metadata:
      labels:
        app: back-end
    spec:
      containers:
        - image: mb923/back-end:v1
          imagePullPolicy: IfNotPresent
          name: back-end
      restartPolicy: Always
```
### Explication : 

- **apiVersion : apps/v1:** Version de l'API Kubernetes utilisée pour ce déploiement.
- **kind: Deployment:** Type d'objet Kubernetes utilisé pour ce déploiement.
- **name: front-end-deployment:** Nom attribué à ce deploiement.
- **replicas: 1:** Nombre de répliques de l'application front-end à exécuter simultanément dans le cluster Kubernetes.
- **image: mb923/front-end:v1** Docker utilisée pour créer les conteneurs de cette application front-end.
- **imagePullPolicy: IfNotPresent** Politique de récupération de l'image Docker. Dans ce cas, si l'image n'est pas déjà présente localement sur le nœud du cluster Kubernetes, elle sera récupérée.
- **restartPolicy: Always** Politique de redémarrage des conteneurs. Cette politique indique à Kubernetes de redémarrer toujours le conteneur si celui-ci échoue ou termine son exécution.


### Création du service kubernetes NodePort : 

(petit texte) 

```
apiVersion: v1
kind: Service
metadata:
  name: front-end-service
spec:
  ports:
    - name: http
      targetPort: 8080 #port du code 
      port: 80 #port du service
  selector:
    app: front-end
  type: NodePort
```

```
apiVersion: v1
kind: Service
metadata:
  name: back-end-service
spec:
  ports:
    - nodePort: 31280
      port: 80         
      protocol: TCP
      targetPort: 8080
  selector:
    app: back-end
  type: NodePort
```

### Explication : 

- **apiVersion: v1:** Version de l'API Kubernetes utilisée pour ce service.
- **kind: Service:** Type d'objet Kubernetes utilisé pour ce service.
- **metadata/name: front-end-service:** Nom attribué à ce deploiement.
- **spec/ports:** Spécifie les ports à ouvrir pour le service.
  - **name: http:** Nom du port. Dans ce cas, il est nommé "http".
  - **targetPort: 8080:** Port cible sur lequel l'application front-end écoute à l'intérieur des pods.
  - **port: 80:** Port sur lequel le service sera accessible à l'intérieur du cluster Kubernetes.
- **selector/app: front-end:** Sélectionne les pods auxquels ce service doit être associé. Dans ce cas, seuls les pods avec le libellé "app: front-end" seront associés à ce service.
- **type: NodePort:** Type de service. Dans ce cas, il est de type "NodePort", ce qui permet d'exposer le service sur un port fixe sur chaque nœud du cluster Kubernetes.

### Définition d'une Gateway Istio :

(petit texte) 

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: microservice-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

### Explication : 

- **apiVersion: networking.istio.io/v1alpha3:** Version de l'API Istio utilisée pour cette passerelle.
- **kind: Gateway:** Type d'objet Istio utilisé pour cette passerelle.
- **metadata/name: microservice-gateway:** Nom  attribué à la passerelle.
- **spec/selector:** Sélectionne le pod Istio Ingress Gateway auquel cette passerelle doit être associée.
  - **istio: ingressgateway:** Sélectionne le pod avec le libellé "istio: ingressgateway".
- **spec/servers:** Définit les paramètres du serveur pour cette passerelle.
  - **port/number: 80:** Numéro de port pour le serveur. Dans ce cas, il est configuré sur le port 80.
  - **port/name: http:** Nom du port. Dans ce cas, il est nommé "http".
  - **port/protocol: HTTP:** Protocole utilisé pour le trafic entrant. Dans ce cas, c'est le protocole HTTP.
  - **hosts: ["*"]:** Liste des hôtes acceptés par cette passerelle. Dans ce cas, il accepte tous les hôtes (wildcard "*"), ce qui signifie qu'il gère le trafic entrant pour tous les hôtes.


### Définition d'un Proxy Istio : 

(petit texte) 

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: back-end-proxy
spec:
  hosts:
  - "*"
  gateways:
  - microservice-gateway
  http:
  - match:
    - uri:
        prefix: /back-end/
    rewrite:
      uri: /
    route:
    - destination:
        port:
          number: 80
        host: back-end-service.default.svc.cluster.local
```

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: front-end-proxy
spec:
  hosts:
  - "*"
  gateways:
  - microservice-gateway
  http:
  - match:
    - uri:
        prefix: /front-end/
    rewrite:
      uri: /
    route:
    - destination:
        port:
          number: 80
        host: front-end-service.default.svc.cluster.local
```

### Explication : 

- apiVersion: networking.istio.io/v1alpha3: Version de l'API Istio utilisée pour ce service virtuel.
- kind: VirtualService: Type d'objet Istio utilisé pour ce service virtuel.
- metadata/name: front-end-proxy: Nom attribué à ce service virtuel.
- spec/hosts: Liste des hôtes pour lesquels ce service virtuel s'applique. Dans ce cas, il s'applique à tous les hôtes avec le wildcard "*".
- spec/gateways: Liste des passerelles à associer à ce service virtuel. Dans ce cas, il est associé à la passerelle "microservice-gateway".
- spec/http: Définit les règles de routage HTTP pour ce service virtuel.
  - match/uri/prefix: /front-end/: Cette règle de correspondance spécifie que ce service virtuel s'applique aux requêtes avec un préfixe d'URI "/front-end/".
  - rewrite/uri: / Cette règle de réécriture modifie l'URI de la requête en "/", ce qui signifie qu'il réécrit l'URI de la requête à partir de "/front-end/" à "/".

- route/destination: Définit la destination des requêtes correspondantes.
  - port/number: 80: Numéro de port vers lequel rediriger le trafic. Dans ce cas, il est configuré sur le port 80.
  - host: front-end-service.default.svc.cluster.local: Nom du service Kubernetes vers lequel rediriger le trafic. Dans ce cas, il est dirigé vers le service "front-end-service" dans l'espace de noms par défaut ("default").


## Déploiement sur Minikube :

Appliquez le fichier YAML pour déployer notre applications sur Minikube : 

```
kubectl apply -f front-back-app.yml
```

## Accéder au service front-end :

Pour obtenir l'URL pour accéder au service front-end déployé sur Minikube , exécuter la commande suivante : 

```
minikube service front-end --url
```

Ouvrir L'URL dans votre navigateur pour accéder à l'application front-end










