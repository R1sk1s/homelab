# Instalación y configuración de Proxmox VE

## Objetivo

La idea es crear un servidor que me permita disponer de una plataforma de virtualización fácil de administrar y que sea estable.

Después de investigar diferentes alternativas, decidí utilizar Proxmox VE porque es una solución open source orientada a la virtualización y encaja bien con lo que busco para este proyecto.

Proxmox será la base de mi homelab, sobre la que iré creando diferentes máquinas virtuales y servicios a medida que avance el laboratorio.

## Hardware utilizado

Para montar el primer nodo del homelab estoy utilizando un Dell OptiPlex 7060 Micro. Elegí un equipo de formato reducido y de segunda mano, ya que permite disponer de un servidor dedicado con un consumo y tamaño contenidos sin realizar una inversión elevada.

### Especificaciones

- **Equipo:** Dell OptiPlex 7060 Micro
- **Procesador:** Intel Core i5-8500T (6 núcleos / 6 hilos)
- **Memoria RAM:** 12 GB DDR4
- **Almacenamiento:** SSD M.2 NVMe de 256 GB
- **Red:** Ethernet Gigabit
- **Virtualización:** Intel VT habilitado

El objetivo inicial no es disponer de una infraestructura con muchos recursos, sino comenzar con el hardware disponible, observar el consumo real de las máquinas virtuales y ampliar los recursos cuando exista una necesidad concreta.

## Decisiones de diseño

Antes de comenzar el proyecto conocía otras soluciones de virtualización como Hyper-V y VMware. Tras investigar diferentes alternativas decidí utilizar Proxmox VE como plataforma principal del homelab.

Buscaba una solución que pudiera instalar directamente en el servidor, que fuese sencilla de administrar y que me permitiera crear y gestionar diferentes máquinas virtuales desde una interfaz web.

Proxmox VE encajaba bien con estos requisitos al ser una plataforma open source basada en Debian y utilizar tecnologías como KVM y LXC para la virtualización.

## Instalación de Proxmox

Proxmox VE se instaló directamente sobre el Dell OptiPlex, utilizando el equipo como servidor dedicado para el homelab.

Durante la instalación decidí utilizar ext4 con LVM como sistema de almacenamiento en lugar de ZFS.

### Elección del almacenamiento

Proxmox permite utilizar diferentes configuraciones de almacenamiento. Una de las opciones que valoré fue ZFS, que ofrece características interesantes como integridad de datos, snapshots y diferentes posibilidades de redundancia.

Sin embargo, el servidor dispone actualmente de un único SSD NVMe de 256 GB y el objetivo de esta primera fase es mantener una infraestructura sencilla y con un consumo de recursos reducido.

Por este motivo decidí utilizar ext4 con LVM. Para el estado actual del laboratorio considero que es una solución más sencilla y suficiente para almacenar las máquinas virtuales.

Si en el futuro incorporo varios discos o un sistema de almacenamiento dedicado, volveré a valorar el uso de ZFS.

## Configuración inicial

Una vez instalado Proxmox VE, configuré el nodo con el hostname `pve01` y una dirección IP estática.

### Configuración de red

- **Hostname:** `pve01`
- **Dirección IP:** `192.168.0.2`
- **Red:** `192.168.0.0/24`
- **Gateway:** `192.168.0.1`

Decidí utilizar una dirección IP fija porque Proxmox es un servidor que necesito poder localizar siempre en la misma dirección.

Si utilizase DHCP, el router podría asignarle una dirección diferente en algún momento, lo que provocaría que la dirección que utilizo para acceder a la interfaz web dejase de funcionar.

Por este motivo, los equipos principales de infraestructura del homelab utilizarán direcciones IP estáticas.

El servidor DHCP del router utiliza actualmente el rango `192.168.0.10 - 192.168.0.250`. Para evitar posibles conflictos de direcciones, he reservado de forma lógica las primeras direcciones de la red para la infraestructura del laboratorio.

| Dirección | Equipo | Función |
| `192.168.0.1` | Router | Gateway de la red |
| `192.168.0.2` | `pve01` | Hipervisor Proxmox |

## Almacenamiento

Después de la instalación, Proxmox dispone de dos almacenamientos principales que utilizo con funciones diferentes:

- **`local`**: almacenamiento basado en directorio. Lo utilizo principalmente para guardar imágenes ISO y otros archivos necesarios para crear las máquinas virtuales.
- **`local-lvm`**: almacenamiento LVM-Thin destinado principalmente a los discos virtuales de las máquinas virtuales.

Esta separación permite mantener los archivos utilizados para desplegar las máquinas separados de sus discos virtuales.

## Red

Para la conectividad del homelab, el servidor Proxmox está conectado mediante Ethernet a la red local `192.168.0.0/24`.

Proxmox utiliza un bridge de red llamado `vmbr0`. Este bridge permite conectar las interfaces de red virtuales de las máquinas virtuales con la interfaz de red física del servidor.

De esta forma, las máquinas virtuales pueden comunicarse con otros dispositivos de la red local como si fueran equipos independientes conectados a la misma red.

La estructura actual es:

Internet
   |
Router (`192.168.0.1`)
   |
Red local (`192.168.0.0/24`)
   |
Dell OptiPlex - Proxmox (`192.168.0.2`)
   |
 `vmbr0`
   |
   +-- `srv-linux-01` (`192.168.0.3`)
   |
   +-- `docker-01` (`192.168.0.4`)

Actualmente todas las máquinas se encuentran en la misma red local. En fases posteriores del proyecto se estudiará la segmentación de la infraestructura mediante VLANs y un firewall dedicado.

## Verificaciones

Una vez finalizada la instalación y configuración inicial realicé varias comprobaciones para asegurarme de que el nodo estaba preparado para empezar a desplegar máquinas virtuales.

Se verificó:

- Acceso correcto a la interfaz web de Proxmox mediante `192.168.0.2`.
- Conectividad del servidor con la red local.
- Configuración correcta de la dirección IP y del gateway.
- Detección correcta del almacenamiento disponible.
- Intel VT habilitado para permitir virtualización por hardware.

Una vez realizadas estas comprobaciones, el nodo `pve01` quedó preparado para comenzar a crear las primeras máquinas virtuales.


## Problemas encontrados

Durante la instalación inicial de Proxmox no se produjeron incidencias relevantes.

Los principales puntos de esta fase estuvieron relacionados con la elección del almacenamiento, la planificación inicial del direccionamiento IP y la preparación del nodo para utilizarlo como base del laboratorio.


## Resultado

El resultado de esta primera fase es un nodo Proxmox completamente funcional que servirá como base de virtualización del homelab.

Sobre este nodo iré desplegando diferentes máquinas virtuales con funciones separadas, intentando mantener cada servicio organizado y evitando instalar aplicaciones directamente sobre el hipervisor cuando no sea necesario.

Actualmente la infraestructura comienza con:

| VM | Hostname | IP | Función |

## Referencias

- Documentación oficial de Proxmox VE