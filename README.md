# TP - Utilisation de Helm

## I- Installer Helm et déployer nextcloud
Afin d'installer Helm j'ai effectuer ces commandes ci-dessous :
```sh
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
```
```sh
sudo apt-get install apt-transport-https --yes
```
```sh
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
```
```sh
sudo apt-get update
```
```sh
sudo apt-get install helm
```
Ensuite j'ajoute le repo nextcloud et j'installe le nextcloud
```sh
helm repo add nextcloud https://nextcloud.github.io/helm/
helm install my-nextcloud nextcloud/nextcloud
```
Afin d'ajouter des replicas je créer un fichier et je modifie les valeurs par défaut (j'ai également ajouter un ingress afin d'y accèder)
```sh
vi values.yml
```
values.yml :
```sh
replicaCount: 3
ingress:
  enabled: true
```
J'upgrade ensuite mon nextcloud afin d'effectuer les modifications
```sh
helm upgrade -f values.yml my-nextcloud nextcloud/nextcloud
```
Je check l'ingress 
```sh
kubectl get ingress
```
Et je met l'host dans mon fichier /etc/hosts, je peux ensuite accèder à mon nextcloud.

## II- Créer sa première chart
Je créé tout d'abord ma chart comme ceci :
```sh
helm create chart01
```
Ensuite je supprime tout ce dont je n'ai pas besoin :
```sh
rm -rf chart01/templates/<fichier sans importance>
```
Je modifie les fichiers _helpers.tl, deployment.yaml, ingress.yaml, service.yaml afin de supprimer les entrées qui ne sont pas utile / qui était en adéquation avec les fichiers supprimés. Puis je créer mon fichier values par défaut :
```sh
vi chart01/values.yaml
```
values.yaml :
```sh
service:
  port: 80
  type: ClusterIP
image:
  repository: nginx
  tag: latest
```
Je créer ensuite mes fichiers values prod et dev
```sh
vi values-prod.yml
```
values-prod.yml :
```sh
image:
  tag: 1.22
ingress:
  hosts:
    - host: srv-prod.test
      paths:
        - path: "/"
          pathType: Prefix
  enabled: true
```
```sh
vi values-dev.yml 
```
values-dev.yml :
```sh
image:
  tag: 1.23
ingress:
  hosts:
    - host: srv-dev.test
      paths:
        - path: "/"
          pathType: Prefix
  enabled: true
```
Je modifier ensuite mon fichier /etc/hosts pour l'ingress
```sh
127.0.0.1 srv-prod.test
127.0.0.1 srv-dev.test
```
J'installe mon serveur de prod :
```sh
helm install -f values-prod.yml -n production --create-namespace srv-prod ../chart01
```
Puis j'installe mon serveur de dev :
```sh
helm install -f values-dev.yml -n developpement --create-namespace srv-dev ../chart01
```
Cette commande me sert à lister ce qui tourne actuellement
```sh
helm list -A
```
On peut les désinstaller comme ceci :
```sh
helm uninstall srv-prod -n production
helm uninstall srv-dev -n developpement
```

## III- ConfigMap
Dans un premier temps je créer mon fichier ConfigMap
```sh
vi chart01/templates/configmap.yaml
```
configmap.yaml :
```sh
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  index: {{ .Values.index | quote }}
```
Ensuite je modifie mon fichier deployment en ajoutant ceci :
```sh
          volumeMounts:
            - name: index
              mountPath: "/usr/share/nginx/html/"
      volumes:
      - name: index
        configMap:
          name: {{ .Release.Name }}-configmap
          items:
          - key: "index"
            path: "index.html"
```
Puis je modifie mes fichiers values en ajoutant ceci
values-dev.yml :
```sh
index: |
  <html>
  <h1>Yo Prod</h1>
  </html>
```
values-prod.yml :
```sh
index: |
  <html>
  <h1>Yo Dev</h1>
  </html>
```
J'applique ensuite mes changements
```sh
helm upgrade -f values-prod.yml -n production srv-prod ../chart01
helm upgrade -f values-dev.yml -n developpement srv-dev ../chart01
```
## IV- Repo Helm
Je vais dans un premier temps créer une branche sur mon git
```sh
git checkout -b gh-pages
git push --set-upstream origin gh-pages
```
Je vais ensuite sur github dans les paramètres de mon repo afin de créer un github pages sur ma branch précédement créé. 
Ensuite, je package mon chart
```sh
helm package chart01/
```
On va ensuite générer le fichier index file à l'aide de cette commande :
```sh
helm repo index . --url https://theofeu.github.io/TP03-Kubernetes/
```
Puis on va pouvoir récupèrer notre repo :
```sh
helm repo add chart01 https://theofeu.github.io/TP03-Kubernetes/
```
Vous pouvez voir votre repo comme ceci :
```sh
helm repo list
```
Nous pouvons ensuite déployer notre propre chart :
```sh
helm install srv-test chart01/chart01
```
