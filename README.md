# DevOps ELK & Web: Monitoreo y Despliegue en KubeSphere
Este proyecto en basado en una prueba profesional para evaluar las habilidades de estudiantes para el area de Seguridad logica-Telconet

Este documento detalla los pasos para la instalación y configuración de un entorno DevOps utilizando ELK Stack y el despliegue de una aplicación web en KubeSphere.
--Kubernetes y ELK Stack:
- Se requiere que se despliegue kubesphere "All-in-One" en la VM proporcionada.
- Una vez desplegado el ambiente de kubernetes, proceder con la instalación de Elastic Cloud on Kubernetes cumpliendo los siguientes puntos:
- Todos los componenetes que se levanten deben estar dentro del namespace "elk-stack-ns".
- La versión del Custom Resource Definition del ELK debe ser la 2.12
- La versión de las imagenes a usar de elasticsearch, kibana y logstash sea 8.13.
- El cluster de elasticsearch debe tener como nombre "elk-cluster".
- Se espera que el Kibana pueda ser accedido desde el puerto 30555 via web

---

## 1. Instalación de la VM para el Servidor

### 1.1 Requisitos de la VM
- **Sistema Operativo**: AlmaLinux 8
- **CPU**: Mínimo 6 vCPUs
- **RAM**: Mínimo 9GB
- **Almacenamiento**: Mínimo 50GB

### 1.2 Configuración de la VM
```bash
# Actualizar paquetes
dnf update -y

# Configurar hostname
hostnamectl set-hostname kubesphere-server
```

---

## 2. Instalación de KubeSphere All-in-One

### 🔧 Requisitos Previos  

Antes de comenzar con la implementación, asegúrate de cumplir con los siguientes requisitos:  

#### 📌 Dependencias del Sistema  

| Dependencia  | Kubernetes ≥ 1.18 | Kubernetes < 1.18 |
|-------------|-------------------|-------------------|
| `socat`     | Obligatorio       | Opcional pero recomendado |
| `conntrack` | Obligatorio       | Opcional pero recomendado |
| `ebtables`  | Opcional pero recomendado | Opcional pero recomendado |
| `ipset`     | Opcional pero recomendado | Opcional pero recomendado |

```bash
sudo dnf install -y socat conntrack-tools ebtables ipset
```

Acceder a la UI de KubeSphere desde `http://<IP-SERVIDOR>:30880`

---

## 3. Despliegue de ELK Stack en KubeSphere

### 3.1 Creación del Namespace
```bash
kubectl create namespace elk-stack-ns
```

### 3.2 Instalación de Elastic Cloud on Kubernetes (ECK)
```bash
kubectl apply -f https://download.elastic.co/downloads/eck/2.12/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.12/operator.yaml
```

### 3.3 Despliegue de Elasticsearch
```bash
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elk-cluster
  namespace: elk-stack-ns
spec:
  version: 8.13.1
  nodeSets:
  - name: default
    count: 3
EOF
```

### 3.4 Despliegue de Kibana
```bash
cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: elk-stack-ns
spec:
  version: 8.13.1
  count: 1
  elasticsearchRef:
    name: elk-cluster
EOF
```

Kibana accesible en `http://<IP-SERVIDOR>:30555`

### 3.5 Despliegue de Logstash
```bash
cat <<EOF | kubectl apply -f -
apiVersion: logstash.k8s.elastic.co/v1alpha1
kind: Logstash
metadata:
  name: logstash-output
  namespace: elk-stack-ns
spec:
  version: 8.13.1
  elasticsearchRefs:
    - clusterName: elk-cluster
      name: elk-cluster
  pipelines:
    - pipeline.id: logstash-output-pipeline
      configMap:
        name: configmap-logstash-output-pipeline
  services:
    - name: logstash-output-service
      service:
        spec:
          type: ClusterIP
          ports:
            - port: 32417
              name: output-port
              protocol: TCP
              targetPort: 32417
EOF
```

---

## 4. Despliegue del Portafolio Web en KubeSphere

### 4.1 Construcción y subida de la imagen Docker
```bash
docker build -t jntobar/mi-portafolio:v1 .
docker push jntobar/mi-portafolio:v1
```

### 4.2 Creación del Namespace para la Aplicación
```bash
kubectl create namespace web-portfolio
```

### 4.3 Despliegue en KubeSphere
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portfolio-deployment
  namespace: web-portfolio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: portfolio
  template:
    metadata:
      labels:
        app: portfolio
    spec:
      containers:
        - name: portfolio
          image: jntobar/mi-portafolio:v1
          ports:
            - containerPort: 80
EOF
```

### 4.4 Creación del Service
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: portfolio-service
  namespace: web-portfolio
spec:
  type: NodePort
  selector:
    app: portfolio
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30001
EOF
```

La aplicación es accesible desde `http://<IP-SERVIDOR>:30001`

---

## 5. Exposición Pública con Ngrok (Opcional)

```bash
ngrok http <IP-SERVIDOR>:30001
```

---

## 6. Verificación del Despliegue

### 6.1 Verificar pods
```bash
kubectl get pods -n elk-stack-ns
kubectl get pods -n web-portfolio
```

### 6.2 Verificar servicios
```bash
kubectl get svc -n elk-stack-ns
kubectl get svc -n web-portfolio
```

---

¡Con esto tienes un entorno funcional con ELK para monitoreo y un portafolio web desplegado en KubeSphere! 🎉


