# Proyecto Backend: API con Múltiples Motores de Base de Datos y Despliegue Multi-entorno

Este proyecto consiste en una API REST construida en **Node.js** con Express, capaz de conectarse dinámicamente a diferentes motores de bases de datos (MongoDB, PostgreSQL y MySQL) a través de una arquitectura basada en drivers. 

Además, el proyecto incorpora flujos completos de **CI/CD** y **Orquestación**, gestionando despliegues automáticos hacia diferentes infraestructuras dependiendo de la rama en la que se trabaje.

---

## Arquitectura de Ramas y Entornos de Despliegue

El proyecto se estructura en 3 ramas principales, cada una con un propósito y entorno de despliegue específico:

### 1. Rama `jenkins` (Despliegue tradicional con Docker Compose y Jenkins)
Esta rama se enfoca en el despliegue hacia un servidor privado (Ej. VPS en DigitalOcean).
- **Herramientas**: Jenkins, Docker, Docker Compose, AWS CLI (para respaldos).
- **Flujo CI/CD**: 
  - Al realizar un push, Jenkins clona el repositorio gracias al `Jenkinsfile`.
  - Construye la imagen Docker del backend.
  - Sube la imagen a Docker Hub (`acordovadev/proyecto-backend:latest`).
  - Despliega el proyecto localmente en el servidor (`/opt/código/cordova-alumno`) usando `docker-compose up -d`.
- **Servicio Adicional**: Incluye un contenedor secundario que usa `amazon/aws-cli` para realizar backups periódicos a un bucket de AWS S3 cada 3 horas.

### 2. Rama `develop` (Despliegue orquestado con Kubernetes y GitHub Actions)
Esta rama está configurada para el despliegue en un clúster de Kubernetes, ideal para escalabilidad y alta disponibilidad.
- **Herramientas**: Kubernetes (k8s), GitHub Actions.
- **Flujo CI/CD**:
  - Al hacer push a `develop`, un flujo de GitHub Actions (`deploy-k8s.yml`) construye y empuja la imagen a Docker Hub.
  - El flujo también contiene pasos para actualizar dinámicamente los manifiestos de k8s en el clúster.
- **Manifiestos en `k8s/`**:
  - `namespace.yaml`: Aislamiento del entorno.
  - `deployment.yaml`: Gestión de Pods y réplicas.
  - `service.yaml`: Exposición del servicio.
  - `configmap.yaml` y `secret.yaml`: Gestión segura de variables de entorno y credenciales (driver de BD, contraseñas).
  - `cronjob-backup.yaml`: Tarea programada en el clúster para enviar backups a AWS S3.

### 3. Rama `modificador` (Cambio Dinámico de Base de Datos)
Esta rama demuestra la flexibilidad del código al permitir cambiar el motor de base de datos en tiempo real mediante GitHub Actions, sin necesidad de tocar el código fuente.
- **Herramientas**: GitHub Actions (`change-driver.yml`).
- **Flujo**: 
  - Se utiliza el trigger `workflow_dispatch` (manual) en GitHub.
  - El usuario ingresa qué base de datos desea activar (`mysql`, `postgres` o `mongo`).
  - El Action modifica el `ConfigMap` o archivo de configuración y redespliega los pods/contenedores aplicando el nuevo driver dinámicamente (`process.env.MY_DATABASE_DRIVER`).

---

## Requisitos Previos

Para ejecutar el proyecto de forma local, necesitarás:
- **Node.js** (v18+)
- **Docker** y **Docker Compose**
- **Kubernetes (Minikube / kubectl)** (Solo si deseas probar el flujo de `develop`)
- Acceso a bases de datos relacionales y no relacionales (Mongo, Postgres, MySQL).

---

## Guía de Despliegue Local

### Opción A: Despliegue con Docker Compose (Flujo Jenkins)

1. Cambia a la rama `jenkins`:
   ```bash
   git checkout jenkins
   ```
2. Revisa las variables en `docker-compose.yml`. Configura el driver que prefieras en `MY_DATABASE_DRIVER` (postgres, mysql, mongo) y sus respectivas credenciales.
3. Levanta los contenedores:
   ```bash
   docker-compose up -d --build
   ```
4. El servicio estará disponible en `http://localhost:3000`.

### Opción B: Despliegue con Kubernetes (Flujo Develop)

1. Cambia a la rama `develop`:
   ```bash
   git checkout develop
   ```
2. Asegúrate de tener un clúster local encendido (ej. `minikube start`).
3. Aplica los manifiestos en orden:
   ```bash
   kubectl apply -f k8s/namespace.yaml
   kubectl apply -f k8s/configmap.yaml
   kubectl apply -f k8s/secret.yaml
   kubectl apply -f k8s/deployment.yaml
   kubectl apply -f k8s/service.yaml
   kubectl apply -f k8s/cronjob-backup.yaml
   ```
4. Revisa los pods:
   ```bash
   kubectl get pods -n <tu-namespace>
   ```
5. Para acceder localmente con minikube:
   ```bash
   minikube service backend-service -n <tu-namespace>
   ```

---

## Endpoints Principales

- `GET /` : Mensaje de bienvenida ("Hola node dynamic").
- `GET /users` : Retorna la lista de usuarios.
- `GET /users/:id` : Retorna un usuario por su ID.
- `POST /users` : Crea un nuevo usuario en la base de datos configurada.

---

## Backups a AWS S3

El sistema de backups automatizado está presente en ambas infraestructuras principales:
- **En Docker-Compose (`jenkins`)**: Como un servicio que ejecuta un `while true` y hace copias usando `aws s3 cp` cada 3 horas.
- **En Kubernetes (`develop`)**: Como un `CronJob` de Kubernetes (`cronjob-backup.yaml`) ejecutado según su sintaxis de cron.

Ambos métodos requieren que inyectes tus credenciales de AWS (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`).
