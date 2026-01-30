# EduGrade Global - Docker y Contenedores

## Indice

1. [Introduccion a Docker](#introduccion-a-docker)
2. [Docker Compose](#docker-compose)
3. [Servicios del Sistema](#servicios-del-sistema)
4. [Red Docker](#red-docker)
5. [Volumenes y Persistencia](#volumenes-y-persistencia)
6. [Health Checks](#health-checks)
7. [Dockerfiles](#dockerfiles)
8. [Comandos Utiles](#comandos-utiles)

---

## Introduccion a Docker

### Que es Docker?

Docker es una plataforma de contenedorizacion que permite empaquetar aplicaciones con todas sus dependencias en unidades estandarizadas llamadas **contenedores**.

```
+--------------------------------------------------+
|              MAQUINA HOST                         |
|                                                   |
|  +-------------+  +-------------+  +----------+  |
|  | Contenedor  |  | Contenedor  |  |Contenedor|  |
|  |   MongoDB   |  |   Neo4j     |  |  Backend |  |
|  |             |  |             |  |          |  |
|  | - mongo:7.0 |  | - neo4j:5.15|  | - node:20|  |
|  | - config    |  | - plugins   |  | - app.js |  |
|  | - data vol  |  | - data vol  |  | - deps   |  |
|  +-------------+  +-------------+  +----------+  |
|         |               |               |        |
|  +--------------------------------------------------+
|  |            DOCKER ENGINE                         |
|  +--------------------------------------------------+
|                                                   |
+--------------------------------------------------+
```

### Beneficios de Docker

| Beneficio | Descripcion |
|-----------|-------------|
| **Aislamiento** | Cada servicio corre en su propio contenedor sin interferir con otros |
| **Portabilidad** | "Funciona en mi maquina" se convierte en "funciona en cualquier maquina" |
| **Reproducibilidad** | El mismo ambiente en desarrollo, testing y produccion |
| **Versionado** | Imagenes taggeadas permiten rollback facil |
| **Escalabilidad** | Facil de escalar horizontalmente |

---

## Docker Compose

### Que es Docker Compose?

Docker Compose es una herramienta para definir y ejecutar aplicaciones multi-contenedor. Con un archivo YAML se describe toda la infraestructura.

### Estructura del docker-compose.yml

```yaml
version: '3.8'

services:
  # Definicion de cada servicio
  mongodb:
    image: mongo:7.0
    ...

  neo4j:
    image: neo4j:5.15-community
    ...

networks:
  # Red compartida entre servicios
  edugrade-network:
    driver: bridge

volumes:
  # Volumenes para persistencia
  mongodb_data:
    driver: local
```

---

## Servicios del Sistema

### 1. MongoDB - Registro Academico (RF1)

```yaml
mongodb:
  image: mongo:7.0
  container_name: edugrade-mongodb
  restart: unless-stopped
  ports:
    - "27017:27017"
  environment:
    MONGO_INITDB_ROOT_USERNAME: admin
    MONGO_INITDB_ROOT_PASSWORD: edugrade2024
    MONGO_INITDB_DATABASE: edugrade
  volumes:
    - mongodb_data:/data/db
    - ./docker/mongodb/init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro
  networks:
    - edugrade-network
  healthcheck:
    test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
    interval: 10s
    timeout: 5s
    retries: 5
```

**Explicacion de parametros:**

| Parametro | Valor | Proposito |
|-----------|-------|-----------|
| `image` | mongo:7.0 | Imagen oficial de MongoDB version 7.0 |
| `container_name` | edugrade-mongodb | Nombre identificable del contenedor |
| `restart` | unless-stopped | Reinicia automaticamente excepto si se detiene manualmente |
| `ports` | 27017:27017 | Mapea puerto del host al contenedor |
| `environment` | MONGO_INITDB_* | Variables de inicializacion de MongoDB |
| `volumes` | mongodb_data, init-mongo.js | Persistencia de datos y script de inicio |

### 2. Neo4j - Trayectorias Academicas (RF3)

```yaml
neo4j:
  image: neo4j:5.15-community
  container_name: edugrade-neo4j
  restart: unless-stopped
  ports:
    - "7474:7474"  # HTTP Browser
    - "7687:7687"  # Bolt Protocol
  environment:
    NEO4J_AUTH: neo4j/edugrade2024
    NEO4J_PLUGINS: '["apoc"]'
    NEO4J_dbms_security_procedures_unrestricted: apoc.*
  volumes:
    - neo4j_data:/data
    - neo4j_logs:/logs
  networks:
    - edugrade-network
  healthcheck:
    test: ["CMD", "wget", "-q", "--spider", "http://localhost:7474"]
    interval: 10s
    timeout: 10s
    retries: 10
    start_period: 30s
```

**Puertos de Neo4j:**

```
+-------------------+
|      Neo4j        |
|                   |
|  +-------------+  |
|  | Browser UI  |  |  <-- Puerto 7474 (HTTP)
|  | http://     |  |      Interfaz web para consultas
|  +-------------+  |
|                   |
|  +-------------+  |
|  | Bolt Driver |  |  <-- Puerto 7687 (Bolt)
|  | bolt://     |  |      Protocolo binario para apps
|  +-------------+  |
+-------------------+
```

### 3. Cassandra - Auditoria y Analitica (RF4/RF5)

```yaml
cassandra:
  image: cassandra:4.1
  container_name: edugrade-cassandra
  restart: unless-stopped
  ports:
    - "9042:9042"
  environment:
    CASSANDRA_CLUSTER_NAME: EduGradeCluster
    CASSANDRA_DC: datacenter1
    CASSANDRA_RACK: rack1
    CASSANDRA_ENDPOINT_SNITCH: GossipingPropertyFileSnitch
    MAX_HEAP_SIZE: 512M
    HEAP_NEWSIZE: 100M
  volumes:
    - cassandra_data:/var/lib/cassandra
    - ./docker/cassandra/init-cassandra.cql:/docker-entrypoint-initdb.d/init.cql:ro
  networks:
    - edugrade-network
  healthcheck:
    test: ["CMD", "cqlsh", "-e", "describe keyspaces"]
    interval: 30s
    timeout: 10s
    retries: 10
```

**Configuracion de Cassandra:**

| Variable | Valor | Proposito |
|----------|-------|-----------|
| `CASSANDRA_CLUSTER_NAME` | EduGradeCluster | Nombre del cluster |
| `CASSANDRA_DC` | datacenter1 | Datacenter logico |
| `CASSANDRA_RACK` | rack1 | Rack dentro del datacenter |
| `CASSANDRA_ENDPOINT_SNITCH` | GossipingPropertyFileSnitch | Estrategia de descubrimiento de nodos |
| `MAX_HEAP_SIZE` | 512M | Memoria maxima de JVM |
| `HEAP_NEWSIZE` | 100M | Memoria para objetos nuevos |

### 4. Redis - Cache y Sesiones (RF2)

```yaml
redis:
  image: redis:7.2-alpine
  container_name: edugrade-redis
  restart: unless-stopped
  ports:
    - "6379:6379"
  command: redis-server --appendonly yes --requirepass edugrade2024
  volumes:
    - redis_data:/data
  networks:
    - edugrade-network
  healthcheck:
    test: ["CMD", "redis-cli", "-a", "edugrade2024", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5
```

**Parametros de Redis:**

| Parametro | Proposito |
|-----------|-----------|
| `--appendonly yes` | Habilita AOF (Append Only File) para persistencia |
| `--requirepass` | Requiere autenticacion |
| `alpine` | Imagen minima basada en Alpine Linux (~5MB) |

### 5. Backend - API REST

```yaml
backend:
  build:
    context: ./backend
    dockerfile: Dockerfile
  container_name: edugrade-backend
  restart: unless-stopped
  ports:
    - "3000:3000"
  environment:
    NODE_ENV: development
    PORT: 3000
    DOCKER_ENV: "true"
    # MongoDB
    MONGODB_URI: mongodb://admin:edugrade2024@mongodb:27017/edugrade?authSource=admin
    # Neo4j
    NEO4J_URI: bolt://neo4j:7687
    NEO4J_USER: neo4j
    NEO4J_PASSWORD: edugrade2024
    # Cassandra
    CASSANDRA_CONTACT_POINTS: cassandra
    CASSANDRA_LOCAL_DC: datacenter1
    CASSANDRA_KEYSPACE: edugrade
    # Redis
    REDIS_HOST: redis
    REDIS_PORT: 6379
    REDIS_PASSWORD: edugrade2024
  depends_on:
    mongodb:
      condition: service_healthy
    neo4j:
      condition: service_healthy
    redis:
      condition: service_healthy
  networks:
    - edugrade-network
  volumes:
    - ./backend:/app
    - /app/node_modules
```

**Variables de conexion a bases de datos:**

Notar que las URIs usan los **nombres de los servicios** (mongodb, neo4j, cassandra, redis) en lugar de localhost. Docker resuelve estos nombres internamente.

```
Host local:     mongodb://localhost:27017
Dentro Docker:  mongodb://mongodb:27017
                         ^^^^^^^
                         Nombre del servicio
```

### 6. Frontend - React SPA

```yaml
frontend:
  build:
    context: ./frontend
    dockerfile: Dockerfile
  container_name: edugrade-frontend
  restart: unless-stopped
  ports:
    - "5173:5173"
  environment:
    VITE_API_URL: http://localhost:3000/api
  depends_on:
    - backend
  networks:
    - edugrade-network
  volumes:
    - ./frontend:/app
    - /app/node_modules
```

---

## Red Docker

### Arquitectura de Red

```
+------------------------------------------------------------------+
|                    edugrade-network (bridge)                      |
|                                                                   |
|  +------------+    +------------+    +------------+               |
|  |  mongodb   |    |   neo4j    |    | cassandra  |               |
|  | 172.20.0.2 |    | 172.20.0.3 |    | 172.20.0.4 |               |
|  +------------+    +------------+    +------------+               |
|        |                 |                 |                      |
|        +-----------------+-----------------+                      |
|                          |                                        |
|                    +-----+-----+                                  |
|                    |   redis   |                                  |
|                    | 172.20.0.5|                                  |
|                    +-----------+                                  |
|                          |                                        |
|                    +-----+-----+                                  |
|                    |  backend  |                                  |
|                    | 172.20.0.6|                                  |
|                    +-----------+                                  |
|                          |                                        |
|                    +-----+-----+                                  |
|                    | frontend  |                                  |
|                    | 172.20.0.7|                                  |
|                    +-----------+                                  |
|                                                                   |
+------------------------------------------------------------------+
```

### Tipos de Redes Docker

| Tipo | Descripcion | Uso en EduGrade |
|------|-------------|-----------------|
| **bridge** | Red aislada entre contenedores | Usada para comunicacion interna |
| host | Comparte red del host | No usada |
| none | Sin red | No usada |
| overlay | Multiples hosts Docker | Para produccion con Swarm |

### Comunicacion entre Servicios

Los servicios se comunican usando sus nombres como hostnames:

```javascript
// backend/src/config/database.js

// MongoDB
await mongoose.connect('mongodb://mongodb:27017/edugrade');
//                              ^^^^^^^
//                              Nombre del servicio

// Neo4j
const driver = neo4j.driver('bolt://neo4j:7687');
//                                  ^^^^^

// Redis
const redis = new Redis({ host: 'redis', port: 6379 });
//                              ^^^^^
```

---

## Volumenes y Persistencia

### Que son los Volumenes?

Los volumenes son el mecanismo de Docker para persistir datos fuera del ciclo de vida del contenedor.

```
SIN VOLUMEN:
+-------------+     docker rm     +-------------+
| Contenedor  |  ------------->   |   PERDIDO   |
| + Datos     |                   |             |
+-------------+                   +-------------+

CON VOLUMEN:
+-------------+     docker rm     +-------------+
| Contenedor  |  ------------->   | Nuevo       |
+------+------+                   | Contenedor  |
       |                          +------+------+
       v                                 |
+------+------+                          v
|   Volumen   |  <------------------------
|   (datos)   |
+-------------+
```

### Volumenes Definidos

```yaml
volumes:
  mongodb_data:    # Datos de MongoDB
    driver: local
  neo4j_data:      # Datos de Neo4j
    driver: local
  neo4j_logs:      # Logs de Neo4j
    driver: local
  cassandra_data:  # Datos de Cassandra
    driver: local
  redis_data:      # Datos de Redis
    driver: local
```

### Mapeo de Volumenes por Servicio

| Servicio | Volumen | Path en Contenedor | Contenido |
|----------|---------|-------------------|-----------|
| MongoDB | mongodb_data | /data/db | Bases de datos |
| Neo4j | neo4j_data | /data | Grafos |
| Neo4j | neo4j_logs | /logs | Logs de Neo4j |
| Cassandra | cassandra_data | /var/lib/cassandra | SSTables |
| Redis | redis_data | /data | AOF y RDB |

### Bind Mounts para Desarrollo

Ademas de volumenes nombrados, usamos bind mounts para desarrollo:

```yaml
backend:
  volumes:
    - ./backend:/app           # Codigo fuente (bind mount)
    - /app/node_modules        # Volumen anonimo para node_modules
```

Esto permite:
- Editar codigo en el host y verlo reflejado inmediatamente
- Mantener node_modules dentro del contenedor (evita conflictos)

---

## Health Checks

### Proposito de Health Checks

Los health checks permiten a Docker verificar si un servicio esta realmente funcionando, no solo si el proceso existe.

```
Estado del Contenedor:
- starting:   Contenedor iniciando, health check no ejecutado aun
- healthy:    Health check pasando
- unhealthy:  Health check fallando
```

### Health Checks por Servicio

**MongoDB:**
```yaml
healthcheck:
  test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
  interval: 10s    # Cada 10 segundos
  timeout: 5s      # Timeout de 5 segundos
  retries: 5       # 5 intentos antes de unhealthy
```

**Neo4j:**
```yaml
healthcheck:
  test: ["CMD", "wget", "-q", "--spider", "http://localhost:7474"]
  interval: 10s
  timeout: 10s
  retries: 10
  start_period: 30s  # Espera 30s antes de iniciar checks
```

**Cassandra:**
```yaml
healthcheck:
  test: ["CMD", "cqlsh", "-e", "describe keyspaces"]
  interval: 30s      # Cassandra es lento, check cada 30s
  timeout: 10s
  retries: 10
```

**Redis:**
```yaml
healthcheck:
  test: ["CMD", "redis-cli", "-a", "edugrade2024", "ping"]
  interval: 10s
  timeout: 5s
  retries: 5
```

### Dependencias con Health Checks

```yaml
backend:
  depends_on:
    mongodb:
      condition: service_healthy   # Espera a que MongoDB este healthy
    neo4j:
      condition: service_healthy   # Espera a que Neo4j este healthy
    redis:
      condition: service_healthy   # Espera a que Redis este healthy
```

Flujo de inicio:

```
1. Docker inicia MongoDB, Neo4j, Redis, Cassandra
2. Health checks comienzan a ejecutarse
3. Cuando MongoDB, Neo4j y Redis estan "healthy"
4. Docker inicia Backend
5. Cuando Backend esta corriendo
6. Docker inicia Frontend
```

---

## Dockerfiles

### Backend Dockerfile

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Instalar dependencias del sistema
# (necesarias para compilar algunos modulos nativos)
RUN apk add --no-cache python3 make g++

# Copiar package files primero (aprovecha cache de Docker)
COPY package*.json ./

# Instalar dependencias
RUN npm install

# Copiar codigo fuente
COPY . .

# Puerto de la aplicacion
EXPOSE 3000

# Comando de inicio
CMD ["npm", "run", "dev"]
```

**Explicacion de capas:**

```
+---------------------------+
|  CMD ["npm", "run", "dev"]|  <- Cambia poco
+---------------------------+
|  COPY . .                 |  <- Cambia frecuentemente
+---------------------------+
|  RUN npm install          |  <- Solo si cambia package.json
+---------------------------+
|  COPY package*.json ./    |  <- Cambia poco
+---------------------------+
|  RUN apk add ...          |  <- Casi nunca cambia
+---------------------------+
|  WORKDIR /app             |  <- Nunca cambia
+---------------------------+
|  FROM node:20-alpine      |  <- Nunca cambia
+---------------------------+
```

### Frontend Dockerfile

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 5173

CMD ["npm", "run", "dev", "--", "--host"]
```

El flag `--host` expone Vite a todas las interfaces, necesario para acceder desde fuera del contenedor.

---

## Comandos Utiles

### Iniciar Servicios

```bash
# Iniciar todos los servicios en background
docker compose up -d

# Iniciar con rebuild de imagenes
docker compose up -d --build

# Iniciar solo algunas dependencias
docker compose up -d mongodb neo4j redis

# Iniciar con logs visibles
docker compose up
```

### Ver Estado

```bash
# Ver contenedores corriendo
docker compose ps

# Ver logs de todos los servicios
docker compose logs

# Ver logs de un servicio especifico
docker compose logs backend

# Seguir logs en tiempo real
docker compose logs -f backend

# Ver ultimas 100 lineas
docker compose logs --tail=100 backend
```

### Detener Servicios

```bash
# Detener todos los servicios
docker compose down

# Detener y eliminar volumenes (CUIDADO: borra datos)
docker compose down -v

# Detener un servicio especifico
docker compose stop mongodb
```

### Ejecutar Comandos

```bash
# Ejecutar comando en contenedor corriendo
docker compose exec backend npm test

# Abrir shell en contenedor
docker compose exec backend sh

# Conectar a MongoDB
docker compose exec mongodb mongosh -u admin -p edugrade2024

# Conectar a Redis
docker compose exec redis redis-cli -a edugrade2024

# Ejecutar CQL en Cassandra
docker compose exec cassandra cqlsh -e "SELECT * FROM edugrade.eventos_auditoria LIMIT 5"
```

### Mantenimiento

```bash
# Reiniciar un servicio
docker compose restart backend

# Reconstruir una imagen
docker compose build backend

# Ver uso de recursos
docker stats

# Limpiar imagenes no usadas
docker image prune

# Limpiar todo (contenedores parados, redes, imagenes)
docker system prune
```

### Troubleshooting

```bash
# Ver eventos de Docker
docker compose events

# Inspeccionar un contenedor
docker inspect edugrade-backend

# Ver configuracion de red
docker network inspect edugrade-global_edugrade-network

# Ver volumenes
docker volume ls

# Inspeccionar volumen
docker volume inspect edugrade-global_mongodb_data
```

---

## Resumen de Puertos

| Servicio | Puerto Host | Puerto Contenedor | Protocolo |
|----------|-------------|-------------------|-----------|
| MongoDB | 27017 | 27017 | TCP (MongoDB Wire) |
| Neo4j Browser | 7474 | 7474 | HTTP |
| Neo4j Bolt | 7687 | 7687 | Bolt |
| Cassandra | 9042 | 9042 | CQL |
| Redis | 6379 | 6379 | RESP |
| Backend | 3000 | 3000 | HTTP |
| Frontend | 5173 | 5173 | HTTP |

---

## Proximos Documentos

- **03-BASES-DE-DATOS.md**: Esquemas y modelos detallados de cada base de datos
- **04-BACKEND-API.md**: Documentacion completa de la API REST
- **05-FRONTEND.md**: Estructura y componentes React
- **06-GUIA-INSTALACION.md**: Como ejecutar el proyecto
