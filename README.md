# Gatemaster-flux-ping-pong

Este repositorio contiene la configuración necesaria para implementar una aplicación de servidor HTTP simple llamada Pong en un clúster de Kubernetes usando Flux CD.

## Prerrequisitos

Antes de comenzar, asegúrese de tener lo siguiente instalado localmente:

- Un Kubernetes cluster (en este experimento, usamos Kind o también puede ser un proveedor de la nube como DigitalOcean)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Flux CLI](https://fluxcd.io/flux/cmd/)
- GitHub Personal Access Token con acceso al repositorio
- [GitHub CLI](https://cli.github.com/)
- [doctl](https://docs.digitalocean.com/reference/doctl/how-to/install/)

## Preparando el repositorio Git

Para preparar su repositorio Git, siga estos pasos:

1. Crea un nuevo directorio e inicializa un repositorio Git:

    ```bash
    mkdir Gatemaster-flux-ping-pong
    cd Gatemaster-flux-ping-pong
    git init
    ```

2. Crear un directorio llamado `manifests`  para guardar los Kubernetes manifests:

    ```bash
    mkdir manifests
    ```

## Creación de manifiestos de implementación y servicio

Cree los siguientes archivos de manifiesto en el directorio `manifests`:

1. **Desplegar los Manifest**: Crear `pong-deployment.yaml`:

    ```yaml
    apiVersion: apps/v1
kind: Deployment
metadata:
  name: pong-server-deployment
  namespace: pong-namespace  
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pong-server
  template:
    metadata:
      labels:
        app: pong-server
    spec:
      containers:
      - name: pong-server
        image: ghcr.io/s1ntaxe770r/pong:e0fb83f27536836d1420cffd0724360a7a650c13
        ports:
        - containerPort: 8080
    ```

2. **Servicio Manifest**: Crear `pong-service.yaml`:

    ```yaml
    apiVersion: v1
kind: Service
metadata:
  name: pong-server-service
  namespace: pong-namespace 
spec:
  selector:
    app: pong-server
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
    ```

3. Agregue y confirme estos archivos en Git:

    ```bash
    git add .
    git commit -m "Añadiendo manifests de despliegue"
    ```

4. Crea el repositorio de GitHub y envía tus cambios:

    ```bash
    gh repo create Gatemaster-flux-ping-pong --source=. --remote=upstream --public
    git push --set-upstream upstream master
    ```

## Inicio de Sesión en DigitalOcean

Si está utilizando DigitalOcean para su clúster de Kubernetes, inicie sesión con doctl:

1. Autenticación con DigitalOcean:

     ```bash
    doctl auth init
    ```

2.  Lista tus clusters de Kubernetes:

     ```bash
    doctl kubernetes cluster list
    ```

## Bootstrapping Flux

Para iniciar Flux y configurarlo para monitorear su repositorio de GitHub:

1.  Cree un clúster Kind o utilice un clúster Kubernetes de DigitalOcean existente:

     ```bash
    kind create cluster --image kindest/node:v1.28.0 --name flux-cluster
    ```

2. Verificar el estado del cluster:

     ```bash
    kubectl cluster-info
     ```

3. Ejecutar comprobaciones previas a la instalación:

    ```bash
    flux check --pre
    ```

4. Exporta tus credenciales de GitHub:

    ```bash
    export GITHUB_TOKEN=<your-token>
    export GITHUB_USER=<your-username>
    ```

5. Ejecute el comando bootstrap:

    ```bash
    flux bootstrap github \
      --owner=$GITHUB_USER \
      --repository=Gatemaster-flux-ping-pong \
      --branch=master \
      --path=clusters/my-cluster
    ```

6. Verifique la instalación de Flux:

    ```bash
    kubectl get pods -n flux-system
    ```

7. Pull los últimos cambios:

    ```bash
    git pull
    ```

## Cambiando de Namespace

Para cambiar el namespace de tu Pong deployment and service: 

1. Crear un nuevo namespace: 

   ```bash
   kubectl create namespace pong-namespace
    ```

2. Modifica tu `pong-service.yaml` y `pong-deployment.yaml` para incluir el campo `manifest`:

   ```bash
   metadata:
       namespace: pong-namespace
    ```

3. Aplicar los cambios:

   ```bash
   kubectl apply -f pong-deployment.yaml -n pong-namespace
   kubectl apply -f pong-service.yaml -n pong-namespace
    ```

4. Verificar los pods dentro del nuevo namespace:

   ```bash
   kubectl get pods -n pong-namespace
    ```

## Automatización de implementaciones con Flux

Para configurar Flux para implementar sus manifests:

1. Definir el recurso `GitRepository`:

    ```bash
    flux create source git pong \
      --url="https://github.com/$GITHUB_USER/Gatemaster-flux-ping-pong" \
      --branch=master \
      --interval=30s \
      --export > ./clusters/my-cluster/pong-source.yaml
    ```

2. Definir el recurso `Kustomization`:

    ```bash
    flux create kustomization pong \
      --target-namespace=default \
      --source=pong \
      --path="./manifests" \
      --prune=true \
      --interval=5m \
      --export > ./clusters/my-cluster/pong-kustomization.yaml
    ```

3. Inspeccionar el manifest generado:

    ```bash
    cat ./clusters/my-cluster/pong-kustomization.yaml
    ```

4. Commit y push los manifests generados:

    ```bash
    git add clusters/
    git commit -m "Añadiendo kustomize manifests"
    git push
    ```

5. Esperando a que Flux sincronice la aplicación:

    ```bash
    flux get kustomizations --watch
    ```

6. Verificar el despliegue:

    ```bash
    kubectl port-forward svc/pong-server-service 8080:80 -n pong-namespace
    curl http://localhost:8080/ping
    ```

## Actualizando la aplicación de Pong con Flux

Para actualizar el despliegue:

1. Editar el archivo `pong-deployment.yaml` para incrementar las replicas:

    ```yaml
    replicas: 3
    ```

2. Guardar, commit, y hacerle push a los cambios:

    ```bash
    git add pong-deployment.yaml
    git commit -m "Escalando el despliegue de pong a 3 replicas"
    git push origin master
    ```

3. Verificar la actualización:

    ```bash
    flux logs
    kubectl get pods -n pong-namespace
    ```

## Solución de problemas

- **Servicio no encontrado**: si encuentra el mensaje `services "pong-server-service" not found`, verifique que el servicio esté correctamente definido e implementado.
- **Problemas de conexión**: asegúrese de que el reenvío de puertos esté configurado correctamente y verifique si la aplicación se está ejecutando correctamente dentro de los pods.
