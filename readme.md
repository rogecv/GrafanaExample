# Instalar Proyecto Jenkins, Nexus y Sonar Dockerizados

Iniciar Minikube porque lo necesitamos: 
```console
minikube start --driver=docker
```
A minikube le indicamos que utilice solo docker

CLonar repositorio: 
[Repositorio GitHub](https://github.com/bellyster/GrafanaExample)

Ir a la línea 13 del archivo docker-compose.yml y cambiar la línea por la ruta de la carpeta actual: 

```yml
%Actual_Path%/jenkins_home:/var/jenkins_home 

```
Tenemos un docker composer para ejecutar. 


```console
docker compose up -d
```

## Acceder al contenedor e instalar kubectl en el interior

Para poder instalar cosas necesitamos entrar al contenedor: 

```console
docker exec -it --user=root jenkinsdocker /bin/bash
```

Se instala Kubectl en el interior de Jenkins: 

Guía de instalación:  https://k8s-docs.netlify.app/en/docs/tasks/tools/install-kubectl/

Ejecutar el comando de instalación: 

```console
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
```

Debemos darle los permisos de ejecución: 

```console
chmod +x ./kubectl
```

Se mueve al directorio de ejecución: 
```console
mv ./kubectl /usr/local/bin/kubectl
```

Comprobar Instalación: 
```console
kubectl version --output=json --client
```

## Acceder a Jenkins 

Ahora se puede acceder a Jenkins desde la ruta de Minikube: 

http://localhost:8080

Instalación inicial. Necesitamos poner la clave inicial: 
Ruta var/jenkins_home/secrets/initialAdminPassword 

Si este archivo no se ha creado en la ruta en cuestión, obtenerlo utilizando los siguientes comandos (dentro del contenedor, donde aún estamos): 
```console
cat var/jenkins_home/secrets/initialAdminPassword
```
Instalar plugins predefinidos. 

Salir del contenedor: 
```console
exit
```
Una vez la instalación este completa. Instalar el plugin Kubernetes

![Alt text](images/PLUGIN.PNG) 

## Conectar instalación de Docker a Minikube 

Visualizar redes presentes en minikube
```console
docker network ls
```

Nos interesa es la red de minikube.

Conectar el contenedor de jenkins a esta red.
```console
docker network connect minikube jenkinsdocker
```

Comprobamos que se ha agregado de manera correcta
```console
docker container inspect jenkinsdocker
```

## Desconectar Docker de Minikube
Cuando terminemos este ejercicio no olvides desconectar este contenedor a de esa red.

Dado que si no inicia el minikube con su red y el contenedor conectado a esa red podria tener problemas a querer iniciar de manera independiente

Comando para desconectarlo de la red

```console
docker network disconnect minikube jenkinsdocker
```

-----
# Integración con Prometheus

Entrar a la carpeta kubernetes-monitor: 

```console
kubectl apply -f jenkins-account.yaml
```

Ver la configuración del cluster: 
```console
kubectl config view
```

Aquí veremos dos campos importantes: 
certificate-authority: tiene la ruta del certificado de autenticación. 
server: Tiene la ruta del cluster minikube 

Esto nos servira para configurar las credenciales de acceso de Jenkins. 
SI intentamos entrar a la ruta del server, veremos que obtenemos error 403: forbidden. 

Ver los servicios: 
```console
kubectl --namespace default get serviceaccount
```

Ejecutar el service-account-token: 
```console
kubectl apply -f jenkins-service-account-token.yaml
```

Obtener el Token: 
```console
kubectl describe secret/jenkins-token-rk2mg
```

## Añadir credenciales a Jenkins:

Ir a Manage Jenkins, Credentials y añadir una credencial global: 
Tipo: Secret text 
Secret: el token que obtuvimos al ejecutar el describe anterior 
Ver nombre del ID en linea 15 jenkins file 
ID: kubernete-jenkis-server-account 
Descripción: kubernetes


## Configurar CLoud en Jenkins: 

Ir a Manage Jenkins, CLouds, New CLoud. 
CLoud Name: kubernetes
tipo: kubernetes. 
Crear. 

![Alt text](<images/Jenkins cloud.PNG>)

Ir a KUbernetes cloud details. El server es la url que nos daba 403. 
Kubernetes URL: 
```console
kubectl config view
```
Certificate key: Necesitamos el certificado que nos da en el mismo comando. Hacer un cat para obtener su valor, recordar que los certificados deben pegarse incluyendo las leyendas de begin and end certificate. 

```console
cat /home/bellyster/.minikube/ca.crt
```

![Alt text](<images/Cloud Configuration 1.PNG>)

Seleccionar las credenciales agregadas anteriormente. 
Y dar click en Test Connection

![Alt text](<images/test conection.PNG>)

## Crear Jenkins Pipeline: 

Ir al panel principal y crear un nuevo job de tipo Pipeline. 
En la sección build, seleccionar from SCM file, y dar el link del repositorio Jenkins: 
LINK: https://github.com/bellyster/GrafanaExample
script path: kubernetes-monitor/Jenkinsfile
Testear proceso. 

![Alt text](images/pipelineSuccess.PNG)

## Entrar al Proyecto Desplegado con Minikube: 

Obtenemos la IP de minikube y accedemos al puerto: 31780
```console
minikube ip
```
Y veremos la aplicación desplegada. 
Datos default estan en application properties. 
user: userdevops
password: devops
![Alt text](images/ProyectoDevops.PNG)

# Monitoreo
En kubernetes-monitor entrar a la carpeta Prometheus 

Verificar si tenemos el namespace monitoring creado.

```console
kubectl get namespace
```

Si no aparece lo debemos crear.

```console
kubectl create namespace monitoring
```

Comprobemos nuevamente que este creado:

```console
kubectl get namespace
````

## Entrar a carpeta Prometheus y aplicar la autorización de prometheus
Entrar a la carpeta prometheus en la consola, y luego a la carpeta monitoring.
Ejecutamos el siguiente comando para aplicar el script de Autorización de prometheus. 

```console
kubectl apply -f authorization-prometheus.yaml
```

Abrir una segunda terminal y abrir dashboard de minikube. Debe hacerce en una segunda terminal ya que dejara la terminal tomada mientras se ejecute el dashboard. 

```console
minikube dashboard --url
```

Ir a la url que retorna este comando y veremos el dashboard: 
![Alt text](<images/DASHBOARD minikube.PNG>)

En la sección de "Config Maps" veremos la configuración. 
En la parte superior podemos seleccionar el namespace a visualizar. 
Deberia haber un config Maps en monitoring. 

```console
kubectl apply -f configmap-prometheus.yaml
```

Ahora veremos el config map creado en el namespace de monitoring. 

Hacer deployment de prometheus: 
```console
kubectl apply -f deployment-prometheus.yaml
```

comprobemos si esta todo dentro del namespace

```console
kubectl get all --namespace=monitoring
```

Y luego ejecutamos el comando

```
minikube ip
```

192.168.49.2:30000

Y deberiamos ver a prometheus en acción. Veremos todas las secciones desplegadas, pero falta añadir la configuración de que vamos a monitorear, por lo que al final de todo, veremos una sección en rojo. 
![Alt text](images/StateMetricsError.PNG)

Ahora vamos a configurar los "kube-state-metric", indicando que se va a monitorear. 

```console
kubectl apply -f kube-state-metrics/
```

## Conectar con SLACK para recibir alertas 

En Slack crear un canal llamado Prometheus
Añadir la app: WebHooks entrantes
![Alt text](<images/Webhook entrantes.PNG>)

En los ajustes de integración seleccionar el canal, y copiar la URL del Webhook habilitado para JSONs. 
![Alt text](<images/slack webhookk link.PNG>)

Ahora en la carpeta de "alert-manager"

Archivo "configmap-alert-manager.yaml" en la **linea 12** debemos cambiar:

slack_api_url : **La url que entregó la app de slack** 

Este archivo describe la integración de alertas. 

Ahora ejecutar un apply sobre esta carpeta. 
```console
kubectl apply -f alert-manager/
```
Podemos ver la alerta configurada en : 

**minikubeip:31000**

Esperar un momento y recibiremos alertas

![Alt text](<images/Alerta recibida.PNG>)

Esta alerta se encuentra definida en el archivo: configmap-prometheus.yaml
En la **linea 11**

# Integración de Grafana 

En el directorio the prometheus/monitoring

```console
kubectl apply -f grafana/
```

Se puede ingresar a grafana en la ruta: 

```console
MinikubeIp:32000
```

```console
minikube service grafana --url -n monitoring
```


![Alt text](images/GRAFANA1.PNG) 
user: admin, password: admin

En el "+" (navbar extremo derecho) agregar un import 

![Alt text](images/Grafana2.PNG) 

Solo es necesario indicar el id: 6417 y damos click en "load"

![Alt text](<images/GRAFANA 3.PNG>)

y en la ultima opción seleccionamos ***prometheus***

![Alt text](<images/grafana 4.PNG>) 

y damos click en "import"

![Alt text](<images/grafana 5.PNG>)

# Algunas Métricas de Grafana
- count by (namespace) (kube_pod_info)
- kube_pod_container_resource_limits


### Recordar sacar Jenkins de la red de minikube! 
