# Servidor de Minecraft

## Objetivo

El objetivo de esta fase es desplegar un servidor de Minecraft dentro del homelab que pueda permanecer disponible de forma continua y al que puedan conectarse otros jugadores desde Internet.

Aunque Minecraft es el servicio utilizado, esta parte del proyecto me permite trabajar con conceptos aplicables a otros servicios:

- Creación y dimensionamiento de una máquina virtual.
- Configuración de red e IP estática.
- Despliegue de servicios mediante Docker.
- Publicación de puertos.
- NAT y acceso desde Internet.
- Configuración de firewall.
- Persistencia de datos.
- Monitorización.
- Análisis y ajuste de recursos.

Para mantener este servicio separado del resto del laboratorio decidí utilizar una máquina virtual dedicada llamada `srv-minecraft-01`.

---

## Arquitectura

El servidor se ejecuta dentro de una máquina virtual Debian alojada en Proxmox.

La comunicación general queda de la siguiente forma:

```text
Internet
   │
   │ TCP 25565
   ▼
Router
   │
   │ Port Forwarding / NAT
   ▼
192.168.0.6
srv-minecraft-01
   │
   ▼
Docker
   │
   ▼
Minecraft Forge
```

Desde Internet las conexiones llegan a la dirección pública del router.

El router redirige el tráfico del puerto utilizado por Minecraft hacia la dirección IP privada de `srv-minecraft-01`.

---

## Creación de srv-minecraft-01

Para el servidor decidí utilizar una máquina virtual independiente.

La configuración de red es:

```text
Hostname: srv-minecraft-01
IP:       192.168.0.6/24
Gateway:  192.168.0.1
```

La máquina está conectada a `vmbr0`, por lo que forma parte de la misma red local que el resto de servidores del homelab.

Utilizar una IP estática es especialmente importante en este caso porque el router necesita conocer permanentemente la dirección interna a la que debe redirigir las conexiones de Minecraft.

Si la dirección cambiase mediante DHCP, la regla de port forwarding podría dejar de apuntar al servidor correcto.

---

## Acceso mediante SSH

La administración de la máquina se realiza mediante SSH.

Al igual que en el resto de servidores del homelab, utilizo autenticación mediante clave pública en lugar de depender del acceso mediante contraseña.

Esto me permite administrar `srv-minecraft-01` desde mis equipos sin exponer SSH directamente a Internet.

El puerto SSH está permitido únicamente desde la red local:

```text
22/tcp → 192.168.0.0/24
```

El acceso administrativo y el acceso de los jugadores se mantienen así como dos servicios independientes.

---

## Despliegue mediante Docker

Decidí ejecutar Minecraft mediante Docker en lugar de instalar directamente todas sus dependencias sobre Debian.

Esto permite mantener el servicio encapsulado y simplifica operaciones como:

- Arrancar y detener el servidor.
- Recrear el servicio.
- Mantener su configuración.
- Separar Minecraft del sistema operativo.
- Automatizar su arranque después de reiniciar la máquina.

La imagen utilizada para ejecutar el servidor es:

```text
itzg/minecraft-server
```

El servicio utiliza el puerto estándar de Minecraft:

```text
25565/tcp
```

Docker publica este puerto en el host para permitir que las conexiones que llegan a `srv-minecraft-01` puedan alcanzar el contenedor.

---

## Persistencia de datos

El mundo y la configuración del servidor no pueden depender del ciclo de vida del contenedor.

Los datos utilizados por Minecraft se encuentran bajo:

```text
/opt/minecraft-modded/data/
```

En este directorio se almacenan elementos como:

- Mundo.
- Configuración del servidor.
- Mods.
- Configuración de Forge.
- Parámetros de Java.

Separar los datos del contenedor permite mantener el mundo aunque el contenedor sea reiniciado o recreado.

Al igual que con los volúmenes utilizados anteriormente en Docker, la persistencia no sustituye a una copia de seguridad.

---

## Forge y servidor moddeado

El servidor comenzó como una instalación más sencilla, pero posteriormente decidí utilizar Forge para poder añadir mods.

La versión utilizada es:

```text
Minecraft 1.20.1
Forge 47.4.10
```

La utilización de mods aumenta los requisitos del servidor respecto a una instalación vanilla, especialmente en memoria y CPU.

También obliga a mantener compatibilidad entre:

```text
Versión de Minecraft
        │
        ├── Forge
        │
        └── Mods
```

Los clientes que se conectan deben utilizar una configuración compatible con la del servidor.

---

## Acceso desde la red local

Dentro de la LAN el servidor puede alcanzarse directamente mediante:

```text
192.168.0.6:25565
```

Esto permite comprobar primero el funcionamiento del servicio dentro de la red local antes de añadir el acceso desde Internet.

Separar ambas pruebas facilita el diagnóstico.

Si Minecraft funciona desde la LAN pero no desde Internet, el problema probablemente se encuentra fuera del propio servicio, por ejemplo en:

- Firewall.
- NAT.
- Port forwarding.
- Dirección IP pública.
- Configuración del router.

---

## Acceso desde Internet

Para permitir conexiones externas configuré una regla de port forwarding en el router.

La regla redirige las conexiones destinadas al puerto:

```text
25565/tcp
```

hacia:

```text
192.168.0.6:25565
```

El flujo queda:

```text
Jugador
   │
   ▼
IP pública del router:25565
   │
   ▼
NAT / Port Forwarding
   │
   ▼
192.168.0.6:25565
   │
   ▼
Contenedor Minecraft
```

Después de configurar la red realicé pruebas desde fuera de la LAN para comprobar que otros jugadores podían conectarse correctamente.

---

## Firewall

`srv-minecraft-01` utiliza UFW para controlar las conexiones entrantes.

Las reglas principales utilizadas son:

```text
22/tcp      → permitido desde 192.168.0.0/24
25565/tcp   → permitido para conexiones de Minecraft
9100/tcp    → permitido únicamente desde 192.168.0.5
```

Cada regla corresponde a una necesidad diferente:

```text
22     → administración mediante SSH
25565  → servicio Minecraft
9100   → monitorización mediante Node Exporter
```

El puerto 9100 no necesita estar disponible para toda la red.

Únicamente `monitoring-01`, con dirección:

```text
192.168.0.5
```

necesita acceder a Node Exporter.

Por este motivo configuré:

```bash
sudo ufw allow from 192.168.0.5 to any port 9100 proto tcp
```

Esto permite aplicar el principio de exponer únicamente los servicios necesarios a los equipos que realmente necesitan utilizarlos.

---

## Arranque automático

El servidor está pensado para permanecer disponible aunque la máquina virtual se reinicie.

Después de realizar una parada completa de `srv-minecraft-01` comprobé que, al volver a iniciar la VM, Docker arrancaba el contenedor de Minecraft automáticamente.

El estado podía comprobarse mediante:

```bash
docker ps
```

Durante el inicio el contenedor aparece inicialmente como:

```text
health: starting
```

y una vez terminado el proceso de arranque:

```text
healthy
```

Esto evita tener que acceder manualmente mediante SSH después de cada reinicio para levantar Minecraft.

---

## Monitorización

`srv-minecraft-01` está integrado en la plataforma de monitorización del homelab.

Node Exporter expone las métricas del sistema a través de:

```text
9100/tcp
```

Prometheus, ejecutándose en `monitoring-01`, recopila estas métricas.

Grafana permite visualizar:

- Uso de CPU.
- Uso de RAM.
- Uso de disco.
- Tráfico de red.
- Estado del servidor.

Esto permite observar el comportamiento de la máquina mientras Minecraft está funcionando y utilizar datos reales para decidir cuándo es necesario modificar los recursos asignados.

---

## Problemas encontrados

### Acceso externo

Durante las pruebas de conectividad fue necesario diferenciar entre acceso desde la LAN y acceso desde Internet.

El servidor funcionaba correctamente utilizando:

```text
192.168.0.6
```

desde la red local.

Para el acceso externo era necesario comprobar además la configuración del router y la red desde la que se realizaban las pruebas.

En una de las pruebas el equipo cliente estaba conectado accidentalmente a una red diferente mediante un hotspot móvil.

Este problema me sirvió para comprobar la importancia de conocer desde qué red se está realizando una prueba antes de interpretar el resultado.

---

### Node Exporter inaccesible

Después de integrar el servidor con Prometheus, `srv-minecraft-01` aparecía como:

```text
DOWN
```

El servidor respondía correctamente al ping, pero:

```bash
curl http://192.168.0.6:9100/metrics
```

no conseguía establecer la conexión.

Node Exporter estaba activo y escuchando correctamente en el puerto 9100.

El problema se encontraba en UFW.

Después de permitir el acceso al puerto únicamente desde `monitoring-01`, Prometheus pudo recopilar las métricas y el servidor pasó a aparecer como:

```text
UP
```

Este problema permitió diferenciar entre:

```text
Conectividad con el host
          │
          ▼
Servicio escuchando
          │
          ▼
Firewall
          │
          ▼
Acceso desde el cliente
```

Que una máquina responda a `ping` no significa que todos sus servicios sean accesibles.

---

## Análisis del consumo de memoria

Después de incorporar el servidor a Grafana observé que la utilización de RAM se encontraba cerca del:

```text
93 %
```

En ese momento la máquina virtual tenía asignados:

```text
4 GB RAM
```

Para comprobar si existía realmente presión de memoria utilicé:

```bash
free -h
```

El resultado mostraba aproximadamente:

```text
RAM total:       3.8 GiB
RAM usada:       3.6 GiB
RAM disponible:  283 MiB
Swap utilizada:  362 MiB
```

La poca memoria disponible y el uso de swap indicaban que la máquina estaba funcionando con poco margen.

---

## Identificación del consumo

Para comprobar qué proceso estaba utilizando la memoria ejecuté:

```bash
ps aux --sort=-%mem | head
```

Java era claramente el principal consumidor, utilizando aproximadamente:

```text
3.3 GB RSS
```

El servidor de Minecraft utiliza un proceso Java, por lo que el siguiente paso fue comprobar la configuración de memoria de la JVM.

El archivo utilizado por Forge se encuentra en:

```text
/opt/minecraft-modded/data/user_jvm_args.txt
```

La configuración era:

```text
-Xmx3G -Xms3G
```

Esto significaba que Java tenía configurado un heap de 3 GB dentro de una VM con únicamente 4 GB de RAM.

El resto de memoria debía ser compartido por Debian, Docker, containerd, Node Exporter, caché y la memoria utilizada por Java fuera del propio heap.

---

## Ampliación de RAM

Antes de modificar el heap de Java decidí aumentar primero la memoria de la máquina virtual.

Desde Proxmox cambié:

```text
4096 MiB
```

por:

```text
6144 MiB
```

Después de arrancar nuevamente la máquina mantuve temporalmente:

```text
-Xms3G -Xmx3G
```

Esto permitía comparar el comportamiento antes de modificar dos variables al mismo tiempo.

Con Minecraft iniciado, el servidor pasó a mostrar aproximadamente:

```text
RAM total:       5.8 GiB
RAM usada:       3.6 GiB
RAM disponible:  2.2 GiB
Swap utilizada:  0 B
```

Java seguía utilizando aproximadamente la misma cantidad de memoria.

Esto confirmó que los 4 GB asignados inicialmente a la VM dejaban demasiado poco margen al resto del sistema.

---

## Ajuste de memoria de Java

Una vez ampliada la VM y comprobado que existía margen, modifiqué:

```text
-Xmx3G -Xms3G
```

por:

```text
-Xms3G -Xmx4G
```

La diferencia entre ambos parámetros es:

```text
-Xms → tamaño inicial del heap
-Xmx → tamaño máximo del heap
```

Con esta configuración Java puede comenzar utilizando un heap de 3 GB y crecer hasta un máximo de 4 GB si la carga de Minecraft lo necesita.

No asigné los 6 GB completos a Java porque la máquina virtual también necesita memoria para el sistema operativo y el resto de procesos.

Después de reiniciar el contenedor, Minecraft volvió a aparecer como:

```text
healthy
```

y el sistema mostraba aproximadamente:

```text
RAM total:       5.8 GiB
RAM usada:       3.2 GiB
RAM disponible:  2.6 GiB
Swap utilizada:  0 B
```

A partir de este punto el objetivo es observar el comportamiento cuando haya varios jugadores conectados antes de realizar nuevas ampliaciones.

---

## Gestión de recursos

Actualmente la máquina tiene asignados:

```text
RAM VM:           6 GB
Heap inicial:      3 GB
Heap máximo:       4 GB
```

El host Proxmox dispone actualmente de 12 GB de RAM física y las únicas máquinas que permanecen encendidas habitualmente son:

```text
monitoring-01
srv-minecraft-01
```

Esto permite dedicar más recursos a Minecraft mientras el resto de máquinas del laboratorio permanecen apagadas.

Está prevista una ampliación del host a 16 GB de RAM, lo que permitirá disponer de mayor margen para ejecutar simultáneamente más máquinas virtuales.

---

## Copias de seguridad

El mundo de Minecraft contiene datos persistentes que no deberían depender únicamente del disco de la máquina virtual.

Una parada de la VM, un snapshot o la persistencia de Docker no sustituyen a una copia de seguridad independiente.

El objetivo es mantener copias periódicas del mundo para poder recuperarlo en caso de:

- Corrupción.
- Error durante una actualización.
- Problemas con mods.
- Eliminación accidental.
- Fallo de la máquina virtual.

La automatización de estas copias forma parte de los siguientes pasos del servidor.

---

## Posibles mejoras

El servidor ya funciona correctamente, pero todavía existen varias mejoras que puedo realizar:

- Automatizar copias de seguridad periódicas del mundo.
- Revisar el consumo cuando haya varios jugadores conectados.
- Crear alertas en Grafana/Prometheus.
- Monitorizar específicamente el proceso o servicio de Minecraft.
- Configurar acceso administrativo remoto mediante VPN.
- Implementar DDNS para evitar depender de una dirección IP pública que pueda cambiar.
- Revisar periódicamente las reglas de firewall.
- Documentar el procedimiento de recuperación del servidor.

---

## Resultado

Al finalizar esta fase dispongo de un servidor Minecraft moddeado ejecutándose dentro de una máquina virtual independiente en Proxmox.

El servidor puede ser utilizado desde la red local y se ha configurado el acceso desde Internet mediante NAT y port forwarding.

Minecraft se ejecuta mediante Docker, mantiene sus datos de forma persistente y arranca automáticamente después de reiniciar la máquina.

El servidor también está integrado con Prometheus y Grafana, lo que permitió detectar un problema real de dimensionamiento de memoria.

A partir de las métricas obtenidas comprobé el estado desde el propio sistema, identifiqué Java como principal consumidor y amplié la VM de 4 GB a 6 GB antes de aumentar el heap máximo de Java de 3 GB a 4 GB.

Esta fase me ha permitido trabajar con el ciclo completo de despliegue y operación de un servicio:

```text
Despliegue
    ↓
Red
    ↓
Acceso externo
    ↓
Seguridad
    ↓
Monitorización
    ↓
Diagnóstico
    ↓
Ajuste de recursos
```

El siguiente objetivo es mejorar la disponibilidad y capacidad de recuperación del servicio mediante copias de seguridad y acceso administrativo seguro.

---

## Referencias

- Documentación de Docker.
- Documentación de la imagen `itzg/minecraft-server`.
- Documentación de Minecraft Forge.
- Documentación de UFW.
- Documentación de Proxmox VE.