Déploiement d'applications frontend Vue.js dans K8S
===================================================

Le cas d'étude
--------------

On a besoin de déployer deux applications frontend développées en Vue.js et packagées avec Webpack. Les deux applications sont déployées dans des conteneurs Nginx et présentent la même arborescence de fichiers une fois packagées.
```
/usr/share/nginx/html/
├── index.html
├── js/
│   └── main.js
└── static/
    └── images/
        ├── *.png
        └── *.svg
```

La première application est une single page application et doit être servie sous `/`. La deuxième application propose des fonctionnalités de routage dynamique et doit être servie sous `/association-ui`. 

Configuration du reverse proxy
------------------------------

App 1 - Pas de modification de l'url. Celle-ci est redirigée telle quelle vers le service. 
```
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: theia-public
spec:
  routes:
  - kind: Rule
    match: Host(`$(THEIA_URL)`) && PathPrefix(`/`)
    services:
    - name: frontend-portal-service
      port: 80
```

App - L'url doit être modifiée avant d'être redirigée vers le service. Le fichier `index.html` se trouve dans `/usr/share/nginx/html/` et non pas dans `/usr/share/nginx/html/association-ui/`. On définit le middleware traeffik stripprefix qui supprime le préfix "/association-ui" afin que l'url envoyée au service pointe bien vers `index.html`.

```
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: frontend-variable-association-ingress
  annotations:
    prefix: /association-ui/         <--Kustomize environment variable $(ASSO_PATH_PREFIX)
spec:
  routes:
  - kind: Rule
    match: Host(`$(THEIA_URL)`) && PathPrefix(`$(ASSO_PATH_PREFIX)`)
    middlewares:
    - name: front-association-strip-prefix
      namespace: theia
    services:
    - name: frontend-variable-association-service
      port: 80
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: front-association-strip-prefix
spec:
  stripPrefix:
    prefixes:
    - "$(ASSO_PATH_PREFIX)"
```

Problème des assets statiques après configuration du proxy
-----------------------------------------------------------

Le .js et les images sont requêtés après le chargement du fichier index.html dans le navigateur via HTTP GET. La requête est construite à l'aide de leur chemin relatif et de l'élément HTML \<base\>. 
> The <base> HTML element specifies the base URL to use for all relative URLs in a document. There can be only one <base> element in a document.
> A document's used base URL can be accessed by scripts with Node.baseURI. If the document has no \<base\> elements, then baseURI defaults to location.href.

Ainsi, sans ajouter de configuration, la deuxième application vas requêter le js de la première application. `<script type="text/javascript" src="js/main.js"></script>` est appelé via la requête GET https://test-theia.osug.fr/js/main.js (qui correspond au JS de la première application). Le JS de la deuxième application se situe sous https://test-theia.osug.fr/association-ui/js/main.js.

Pour corriger le problème deux pistes:
 - Spécifier l'élément \<base\>
 - Utiliser la directive [publicPath](https://webpack.js.org/guides/public-path/) de la configuration Webpack. 

La deuxième solution est choisie car elle permet plus facilement d'utiliser la même image pour différent environnement (production, developpement). Une variable d'environnement est substituée au lancement du conteneur et est affecté à la variable gloable webpack `__webpack_public_path__`

L'effet du stripPrefix traeffik est finalement annulé par le publicPath.

Problème avec le routage dynamique
----------------------------------

La deuxième application fonctionne avec du routage dynamique. Un routeur javascript permet de basculer entre les différentes vues de l'application. Pour que le routage fonctionne dans l'application, les routes doivent être préfixées par le prefix utilisé dans le reverse proxy. La variable d'environnement utilisée pour le publicPath est réutilisée.

La configuration de l'application et la configuration du reverse proxy est définie par la variable d'environnement Kustomize (qui est passé au conteneur via le fichier de déploiement).


