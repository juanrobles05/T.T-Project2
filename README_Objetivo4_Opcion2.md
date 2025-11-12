# Reto 4 – Opción 2: Despliegue de Microservicios en Amazon EKS

**Estudiante(s):** Sara Pineda, Juan Diego Robles, Santiago Betancur  
**Profesor:** Álvaro Enrique Ospina  

---

## 1. Descripción de la actividad

Esta actividad consiste en desplegar una aplicación basada en microservicios en un clúster de Kubernetes utilizando el servicio **Amazon Elastic Kubernetes Service (EKS)**.  
El objetivo principal fue demostrar el conocimiento en la creación, configuración y gestión de un entorno distribuido en la nube, aplicando conceptos de contenedores, orquestación y balanceo de carga.

---

## 1.1. Aspectos desarrollados de la actividad

- Creación y configuración de un clúster EKS en AWS con sus respectivos grupos de nodos EC2.  
- Configuración del acceso mediante AWS CLI y `kubectl`.  
- Despliegue de la aplicación **Sock Shop**, compuesta por múltiples microservicios interconectados.  
- Exposición del servicio principal (`front-end`) mediante un **LoadBalancer** público.  
- Validación del correcto funcionamiento a través del acceso al dominio generado por AWS ELB.  
- Documentación completa del proceso, arquitectura y evidencias.

---

## 1.2. Aspectos no desarrollados

- No se integró una base de datos externa gestionada mediante Amazon RDS.  
- No se implementaron métricas de observabilidad o monitoreo avanzado (CloudWatch o Prometheus).

---

## 2. Justificación de la elección de la aplicación

La aplicación **Sock Shop** fue seleccionada porque es una aplicación de referencia ampliamente utilizada para demostrar arquitecturas de microservicios y despliegues en entornos de Kubernetes y nubes públicas como Amazon EKS.  

Sock Shop fue desarrollada por **Weaveworks** y **Container Solutions** con el propósito de servir como un ejemplo educativo y práctico de cómo estructurar, desplegar y escalar sistemas distribuidos en contenedores.  
La aplicación simula una tienda en línea con funcionalidades reales, como catálogo de productos, autenticación de usuarios, carrito de compras, pedidos y procesamiento de pagos.  
Cada una de estas funcionalidades está implementada como un microservicio independiente.

Esta aplicación resulta ideal para el proyecto porque permite:

- Analizar la comunicación entre servicios mediante APIs internas.  
- Comprobar el funcionamiento del balanceo de carga, la escalabilidad y la tolerancia a fallos en Kubernetes.  
- Visualizar cómo Kubernetes gestiona pods, servicios, réplicas y Load Balancers.  
- Practicar la automatización del despliegue en un entorno realista, similar a sistemas empresariales.  

Por estas razones, **Sock Shop** constituye un caso de estudio adecuado para demostrar el despliegue de microservicios en la nube utilizando las herramientas de AWS (EKS, EC2 y ELB), validando conceptos de orquestación, resiliencia y administración de infraestructura basada en contenedores.

---

## 3. Información general de diseño y arquitectura

La aplicación **Sock Shop** representa una tienda en línea implementada bajo una arquitectura de microservicios.  
Cada funcionalidad (usuarios, pedidos, pagos, catálogo, carrito, etc.) se implementa como un microservicio independiente que se comunica a través de servicios internos (`ClusterIP`) dentro del clúster.

### Componentes principales

- **Frontend:** Servicio de entrada expuesto al público mediante un LoadBalancer.  
- **Backend Services:** Microservicios independientes para catálogo, usuarios, pedidos, pagos y envíos.  
- **Databases:** Contenedores que simulan bases de datos para cada servicio (MongoDB, MySQL, Redis).  
- **Mensajería:** RabbitMQ para manejo de colas.

### Arquitectura utilizada

- Kubernetes con namespace `sock-shop`.  
- Servicios de tipo `ClusterIP` para comunicación interna.  
- Servicio `LoadBalancer` (AWS ELB) para exposición externa.  
- Balanceo de carga automático gestionado por AWS.  
- Patrón de microservicios desplegado sobre contenedores Docker.

---

## 4. Descripción del ambiente de desarrollo y técnico

### Lenguaje y herramientas

- **Kubernetes** versión 1.33  
- **AWS CLI** versión 2.15  
- **kubectl** versión 1.30  
- **Amazon EKS** (Elastic Kubernetes Service)  
- **Amazon EC2** (instancias t3.medium, 20 GiB)  
- **Sistema operativo:** Amazon Linux 2023  
- **Docker** versión 24.0  
- **Repositorio base:** [https://github.com/Saraapl/microservicios](https://github.com/Saraapl/microservicios)

---

### Compilación y ejecución

1. **Configurar AWS CLI:**
   ```bash
   aws configure
Actualizar la configuración del clúster:

aws eks update-kubeconfig --region us-east-1 --name bookstore-cluster


Aplicar los manifiestos:

kubectl apply -f complete-demo.yaml


Exponer el frontend:

kubectl edit svc front-end -n sock-shop


Cambiar:

type: NodePort


por:

type: LoadBalancer

Detalles técnicos

Región AWS: us-east-1

Clúster: bookstore-cluster

Namespace: sock-shop

Servicio público: front-end

LoadBalancer generado automáticamente:
a8f516e41a68744099fd775f94b74956-261273995.us-east-1.elb.amazonaws.com

Configuración de parámetros

Puerto de aplicación: 80

Conexión interna: mediante servicios Kubernetes (ClusterIP).

Comunicación entre pods: gestionada por kube-proxy.

Estructura del proyecto
microservices-demo/
│
├── deploy/
│   ├── kubernetes/
│   │   ├── complete-demo.yaml
│   │   ├── catalogue-db-dep.yaml
│   │   ├── user-db-dep.yaml
│   │   └── front-end-svc.yaml
│   └── docker-compose/
│       └── docker-compose.yml
└── README.md

5. Descripción del ambiente de ejecución (en producción)

Plataforma: Amazon Web Services (AWS)

Servicios utilizados

EKS (orquestador de contenedores)

EC2 (nodos del clúster)

Elastic Load Balancer (exposición pública)

Dominio público de acceso
http://a8f516e41a68744099fd775f94b74956-261273995.us-east-1.elb.amazonaws.com

Modo de lanzamiento del servidor

El servicio front-end expone la aplicación en el puerto 80, gestionado por el LoadBalancer creado automáticamente.
El tráfico se enruta al frontend, que se comunica con los microservicios internos a través de servicios Kubernetes.

Guía de uso

Acceder al dominio público desde un navegador web.

Visualizar la interfaz de la tienda Sock Shop.

Explorar el catálogo, agregar productos al carrito y simular compras.

6. Información adicional y conclusiones

El despliegue de la aplicación Sock Shop en Amazon EKS permitió:

Comprender la estructura y funcionamiento de una arquitectura de microservicios.

Implementar orquestación de contenedores con Kubernetes.

Observar el uso de balanceadores de carga y servicios administrados por AWS.

Desarrollar competencias en despliegue automatizado y escalabilidad en la nube.

Conclusión: 

La práctica permitió demostrar el conocimiento técnico en la administración de clústeres Kubernetes en AWS, así como la capacidad de desplegar aplicaciones distribuidas en entornos productivos y escalables.

7. Referencias

Sock Shop – Microservices Demo Repository:
https://github.com/microservices-demo/microservices-demo

AWS Documentation – Amazon EKS:
https://docs.aws.amazon.com/eks/latest/userguide/

Kubernetes Documentation:
https://kubernetes.io/docs/home/

Curso ST0263 – EAFIT
