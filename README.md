# Homelab

Homelab personal creado para practicar administración de sistemas, virtualización, redes, contenedores, monitorización y seguridad en un entorno real.

El proyecto está construido sobre un Dell OptiPlex 7060 Micro utilizando Proxmox VE como hipervisor. A partir de esta base voy desplegando diferentes máquinas virtuales y servicios mientras documento las decisiones tomadas, los problemas encontrados y las soluciones aplicadas.

El objetivo no es únicamente desplegar servicios, sino entender cómo funcionan y utilizar el laboratorio como entorno de aprendizaje y pruebas.

---

## Infraestructura

### Host

```text
Dell OptiPlex 7060 Micro
Intel Core i5-8500T
12 GB RAM
256 GB NVMe
Proxmox VE
```

Está prevista una ampliación a 16 GB de RAM para disponer de mayor margen al ejecutar varias máquinas virtuales simultáneamente.

### Máquinas virtuales

| Máquina | IP | Función | Estado habitual |
|---|---|---|---|
| `srv-linux-01` | `192.168.0.3` | Servidor Linux base y laboratorio | Apagado |
| `docker-01` | `192.168.0.4` | Host dedicado a contenedores | Apagado |
| `monitoring-01` | `192.168.0.5` | Prometheus + Grafana | Encendido |
| `srv-minecraft-01` | `192.168.0.6` | Servidor Minecraft Forge | Encendido |

Todas las máquinas utilizan Debian y están conectadas a la LAN mediante el bridge `vmbr0` de Proxmox.

---

## Arquitectura actual

```text
                              Internet
                                 │
                                 ▼
                               Router
                          192.168.0.1/24
                                 │
                                 ▼
                               Switch
                                 │
                                 ▼
                       Dell OptiPlex 7060
                            Proxmox VE
                          192.168.0.2
                                 │
                               vmbr0
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
          │                      │                      │
          ▼                      ▼                      ▼
   srv-linux-01              docker-01           monitoring-01
   192.168.0.3              192.168.0.4          192.168.0.5
                                                    │
                                             Docker Compose
                                                    │
                                          ┌─────────┴─────────┐
                                          ▼                   ▼
                                     Prometheus             Grafana
                                          │
                                          │ TCP 9100
                                          │ métricas
                                          │
          ┌───────────────────────────────┘
          │
          ▼
   srv-minecraft-01
     192.168.0.6
          │
          ├── Node Exporter :9100
          │
          └── Docker
                │
                ▼
          Minecraft Forge
              :25565
                │
                │ NAT / Port Forwarding
                ▼
             Internet
```

Las cuatro máquinas virtuales se ejecutan de forma independiente dentro de Proxmox y están conectadas a la red local mediante `vmbr0`.

`monitoring-01` centraliza la monitorización mediante Prometheus y Grafana. Prometheus consulta por red las métricas expuestas por Node Exporter en los servidores monitorizados.

Actualmente `srv-minecraft-01` expone Node Exporter en el puerto `9100`, cuyo acceso mediante firewall está permitido únicamente desde `monitoring-01`.

El servidor de Minecraft se ejecuta de forma independiente en `srv-minecraft-01`. El puerto `25565/tcp` se publica mediante Docker y el router redirige las conexiones externas hacia esta máquina mediante NAT/port forwarding.

La arquitectura irá evolucionando a medida que se incorporen nuevos servicios, segmentación de red, copias de seguridad y acceso remoto seguro.

## Tecnologías utilizadas

- Proxmox VE
- Debian
- Docker
- Docker Compose
- Prometheus
- Grafana
- Node Exporter
- Nginx
- SSH
- Git / GitHub
- UFW

---

## Documentación

El proyecto está dividido en diferentes fases.

### 01 - Proxmox

Instalación y configuración inicial del hipervisor, almacenamiento, red y creación de la infraestructura base.

[Ver documentación](docs/01-proxmox/README.md)

### 02 - Servidor Linux

Creación del primer servidor Debian, configuración de red, DNS, SSH mediante claves y preparación de una máquina base.

[Ver documentación](docs/02-servidor-linux/README.md)

### 03 - Docker

Creación de una máquina dedicada a Docker y pruebas con imágenes, contenedores, redes, publicación de puertos, persistencia y Docker Compose.

[Ver documentación](docs/03-docker/README.md)

### 04 - Monitorización

Despliegue de Prometheus, Grafana y Node Exporter para centralizar la monitorización de los servidores.

Incluye la creación de un dashboard reutilizable y un caso real de diagnóstico y redimensionamiento de recursos.

[Ver documentación](docs/04-monitorización/README.md)

### 05 - Minecraft

Despliegue de un servidor Minecraft Forge mediante Docker, configuración de acceso desde Internet, firewall, monitorización y ajuste de recursos.

[Ver documentación](docs/05-minecraft/README.md)

---

## Estado actual

Actualmente el homelab dispone de:

- Hipervisor Proxmox operativo.
- Máquinas virtuales Debian con direccionamiento estático.
- Administración mediante SSH con claves.
- Host dedicado a Docker.
- Servicios desplegados mediante Docker Compose.
- Monitorización centralizada con Prometheus y Grafana.
- Dashboard para CPU, RAM, disco, red y disponibilidad.
- Servidor Minecraft moddeado accesible desde Internet.
- Firewall configurado para limitar el acceso a servicios internos.

---

## Próximos pasos

El laboratorio seguirá creciendo progresivamente. Algunos de los siguientes objetivos son:

- Automatizar copias de seguridad.
- Configurar alertas de monitorización.
- Implementar acceso remoto mediante VPN.
- Configurar DDNS para servicios externos.
- Mejorar la segmentación de red.
- Introducir VLANs.
- Evaluar un firewall dedicado como OPNsense o pfSense.
- Introducir automatización mediante Ansible.
- Crear un laboratorio aislado orientado a ciberseguridad.

---

## Objetivo del proyecto

Este repositorio sirve como documentación del proceso de construcción del homelab.

Además de registrar la configuración final, intento documentar:

- Por qué se toma cada decisión.
- Qué alternativas existen.
- Qué problemas aparecen durante la implementación.
- Cómo se diagnostican.
- Qué solución se aplica y por qué.

La intención es que el laboratorio evolucione junto con mis conocimientos y sirva como entorno práctico para administración de sistemas y ciberseguridad.
