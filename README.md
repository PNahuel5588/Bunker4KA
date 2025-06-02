# 🛡️ Bunker 4 – Plataforma EvolutionAPI + N8N + Portainer

Implementación auto-contenida en **Proxmox** (LXC) que expone —de forma segura— los siguientes
servicios Dockerizados:

| Servicio     | Imagen                         | Puerto interno | Puerto host¹ | Función                      |
|--------------|-------------------------------|---------------:|-------------:|------------------------------|
| EvolutionAPI | `atendai/evolution-api:2.1`   | 8080           | *(túnel)*    | API WhatsApp inteligente     |
| N8N          | `n8nio/n8n:latest`            | 5678           | 5678         | Low-code automation          |
| Redis        | `redis:latest`                | 6379           | —            | Broker de colas / cache      |
| PostgreSQL   | `postgres:14`                 | 5432           | —            | Persistencia EvolutionAPI    |
| pgAdmin 4    | `dpage/pgadmin4:latest`       | 80/443         | 5433 (TLS)   | GUI DB                       |
| Portainer CE | `portainer/portainer-ce`      | 9000/9443      | 9000/9443    | UI gestión contenedores      |

> ¹ **Nunca** exponemos puertos públicos; todo viaja por *SSH local-port-forward*.

---

## 📑 Tabla de contenidos

- [🛡️ Bunker 4 – Plataforma EvolutionAPI + N8N + Portainer](#️-bunker-4--plataforma-evolutionapi--n8n--portainer)
  - [📑 Tabla de contenidos](#-tabla-de-contenidos)
  - [Arquitectura](#arquitectura)
  - [Prerrequisitos](#prerrequisitos)
  - [Despliegue rápido](#despliegue-rápido)
    - [1 – Clona este repo dentro del contenedor LXC](#1--clona-este-repo-dentro-del-contenedor-lxc)
    - [2 – Ajusta variables](#2--ajusta-variables)
    - [3 – Arranca el stack](#3--arranca-el-stack)
    - [4 – Verifica](#4--verifica)
  - [Persistencia](#persistencia)
  - [Alternativas al túnel](#alternativas-al-túnel)
  - [Variables de entorno](#variables-de-entorno)
  - [Comandos útiles](#comandos-útiles)
  - [Problemas conocidos](#problemas-conocidos)
  - [Mantenimiento](#mantenimiento)
    - [Backup DB](#backup-db)
    - [Actualizar imágenes](#actualizar-imágenes)
    - [Rotar API key](#rotar-api-key)
    - [Túneles y acceso seguro](#túneles-y-acceso-seguro)
  - [Licencia](#licencia)

---

## Arquitectura

![Topología Bunker 4](docs/images/4aeb5ac9-619d-4ea8-ab63-59442cf9dea3.jpeg)

| Capa | Componente | Descripción |
|---|------------|-------------|
| 1 | **Modem Movistar** | Nateo ssh :10224 (ssh), :80 y 443 a Router Mkt |
| 2 | **Router mikrotik** | Nateo ssh :10224 al Bastión, 80 y 443 al Nginx Proxy Manager |
| 3 | **Proxmox Host** | Contiene LXC `kevin01` (`10.10.153.4`), container `bastión` y container `nginx-proxy-manager`|
| 4 | **Bastion** | Container con Archlinux - usuario kevin sin privilegios. Puente para túnel hacia Kevin01 |
| 4 | **Nginx-Proxy-Manager** | Container con Archlinux - Proxy Forward del 80 a Container Kevin01 |
| 5 | **Container Kevin01** | Container final, usuario kevin con privilegios |
| 6 | **Docker Stack** | Adentro de kevin01. Servicios listados arriba sobre una red interna. |

---

## Prerequisitos

| Recurso | Versión mínima |
|---------|----------------|
| Ubuntu 22.04 LTS (VM/LXC) | — |
| Docker Engine | `24.x` |
| Docker Compose Plugin | `v2.27+` |
| Acceso SSH bastion | Puerto `10224` |
| DNS → Nginx-Proxy-Manager (opcional) | `*.perfeccion.ar` |

## Despliegue rápido

### 1 – Clona este repo dentro del contenedor LXC

    git clone https://github.com/<ORG>/<REPO>.git
    cd <REPO>

### 2 – Ajusta variables

    cp .env.example .env && nano .env

### 3 – Arranca el stack

    docker compose pull
    docker compose up -d

### 4 – Verifica

    docker compose ps

## Persistencia

evolution_store y evolution_instances → datos WhatsApp

postgres_data → base EvolutionAPI

> Ejemplo: forward local 50000 → contenedor 22

```shell
ssh -N -p 10224 \
    -L 127.0.0.1:50000:10.10.153.4:22 \
    kevin@bunker4.perfeccion.ar
```

```shell
Servicio	 Puerto local	 Destino interno
SSH LXC	        50000	       10.10.153.4:22
N8N	            50001	       10.10.153.4:5678
EvolutionAPI	50002	       10.10.153.4:8080
Portainer	    50003	       10.10.153.4:9000
```

Cliente local → `ssh -p 50000 kevin@localhost` o `http://localhost:50002`

## Alternativas al túnel

```shell
Método	        Ventaja	                           Nota
Cloudflared	    DNS *.trycloudflare.com	        cloudflared tunnel create + YAML
ngrok free	    Fácil, 1 túnel simultáneo	    subdominio cambia en cada restart
```

## Variables de entorno

Ver con `env`

```text
AUTHENTICATION_API_KEY=<clave_hex_64>
TZ=America/Argentina/Buenos_Aires

POSTGRES_USER=postgres
POSTGRES_PASSWORD=<pass_db>
POSTGRES_DB=evolution

PGADMIN_DEFAULT_EMAIL=admin@local
PGADMIN_DEFAULT_PASSWORD=<pass_pgadmin>
Generar clave: openssl rand -hex 32
```

## Comandos útiles

```text
Acción	                 Comando
Listar contenedores	     docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
Logs EvolutionAPI	     docker logs -f evolution_api
Entrar al contenedor	 docker exec -it evolution_api sh
Reiniciar Portainer  	 docker restart portainer
Reset credenciales N8N	 docker stop n8n && docker rm n8n && rm -rf ./n8n_data && docker compose up -d n8n
Métricas live	         docker stats / htop
```

## Problemas conocidos

```shell
Síntoma	                                Causa	                                        Fix
port 8080 already allocated	            8080 ya en uso	docker ps       docker stop <ID>, cambiar puerto compose
EvolAPI reinicia (P1000)	            Credenciales DB incorrectas     Revisar .env y reiniciar EvolutionAPI
permission denied /var/run/docker.sock  Usuario fuera del grupo docker  sudo usermod -aG docker $USER && newgrp docker
Error snap (ngrok)	                    LXC sin squashfs	            Instalar binario .tgz en /usr/local/bin
```

## Mantenimiento
=======
## Ejemplo: forward local 50000 → contenedor 22
ssh -N -p 10224 \
    -L 127.0.0.1:50000:10.10.153.4:22 \
    kevin@bunker4.perfeccion.ar
Servicio	Puerto local	Destino interno
SSH LXC	50000	10.10.153.4:22
N8N	50001	10.10.153.4:5678
EvolutionAPI	50002	10.10.153.4:8080
Portainer	50003	10.10.153.4:9000

Cliente local → ssh -p 50000 kevin@localhost o http://localhost:50002.
---
# Alternativas

Método	Ventaja	Nota
Cloudflared	DNS *.trycloudflare.com	cloudflared tunnel create + YAML
ngrok free	Fácil, 1 túnel simultáneo	subdominio cambia en cada restart
---
# Variables de entorno
AUTHENTICATION_API_KEY=<clave_hex_64>
TZ=America/Argentina/Buenos_Aires

### Backup DB

    docker exec postgres_db pg_dump -U postgres evolution > backups/evolution_$(date +%F).sql

### Actualizar imágenes

    docker compose pull && docker compose up -d

### Rotar API key

    openssl rand -hex 32  # editar .env
    docker compose restart evolution_api

### Túneles y acceso seguro

Crear túnel SSH (VM → Bastion → Contenedor)

PGADMIN_DEFAULT_EMAIL=admin@local
PGADMIN_DEFAULT_PASSWORD=<pass_pgadmin>
Generar clave: openssl rand -hex 32
---
# Comandos útiles
**Acción                   	Comando
Listar contenedores 	docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
Logs EvolutionAPI	    docker logs -f evolution_api
Entrar al contenedor	docker exec -it evolution_api sh
Reiniciar Portainer	    docker restart portainer
Reset credenciales N8N	docker stop n8n && docker rm n8n && rm -rf ./n8n_data && docker compose up -d n8n
Métricas live	        docker stats / htop**
---
# Solución de problemas
**Síntoma	Causa	Fix
port 8080 already allocated	8080 ya en uso	docker ps, docker stop <ID>, cambiar puerto compose
EvolAPI reinicia (P1000)	Credenciales DB incorrectas	Revisar .env y reiniciar EvolutionAPI
permission denied /var/run/docker.sock	Usuario fuera del grupo docker	sudo usermod -aG docker $USER && newgrp docker
Error snap (ngrok)	LXC sin squashfs	Instalar binario .tgz en /usr/local/bin**
---
# Mantenimiento
## Backup DB
**docker exec postgres_db pg_dump -U postgres evolution > backups/evolution_$(date +%F).sql
**---
## Actualizar imágenes
docker compose pull && docker compose up -d
---
## Rotar API key
openssl rand -hex 32  # editar .env
docker compose restart evolution_api
## Licencia

MIT © Copyleft 2025 Bunker 4
