# Monitorización del homelab

## Objetivo

El objetivo de esta fase es disponer de un sistema centralizado desde el que poder comprobar el estado y el consumo de recursos de los servidores del homelab.

Hasta este momento podía comprobar manualmente el estado de cada máquina mediante SSH y comandos como `free`, `top` o `df`, pero a medida que aumenta el número de servidores resulta más útil disponer de un punto central desde el que consultar esta información.

Para ello decidí crear una máquina virtual dedicada llamada `monitoring-01` y utilizar Prometheus y Grafana.

La idea es poder visualizar métricas como:

- Uso de CPU.
- Uso de RAM.
- Uso de disco.
- Tráfico de red.
- Estado de los servidores.

Además de visualizar las métricas, quería entender cómo se obtienen los datos y cómo se comunican entre sí los diferentes componentes del sistema de monitorización.

---

## Arquitectura de monitorización

La arquitectura utilizada está formada por tres componentes principales:

- **Node Exporter:** obtiene y expone métricas del sistema operativo.
- **Prometheus:** consulta periódicamente esas métricas y las almacena.
- **Grafana:** utiliza Prometheus como fuente de datos y permite representarlos mediante dashboards.

El flujo de información es:

```text
Servidor
   │
   │ Node Exporter :9100
   ▼
Prometheus
   │
   │ consultas PromQL
   ▼
Grafana
```

Node Exporter no envía directamente los datos a Prometheus.

Es Prometheus quien realiza periódicamente peticiones a los servidores configurados para obtener sus métricas. Este funcionamiento se conoce como modelo **pull** o **scraping**.

---

## Preparación de monitoring-01

Para separar la monitorización del resto de servicios decidí utilizar una máquina virtual dedicada.

La máquina se configuró como:

```text
Hostname: monitoring-01
IP: 192.168.0.5
RAM: 2 GB
```

Al igual que otras máquinas del laboratorio, utiliza Debian y una dirección IP estática.

Separar la monitorización en una máquina propia permite mantener este servicio independiente de las máquinas que se están monitorizando.

Por ejemplo, si el servidor de Minecraft tiene un problema, la plataforma de monitorización puede continuar funcionando y permitir analizar lo ocurrido.

---

## Despliegue de Prometheus y Grafana

Prometheus y Grafana se desplegaron mediante Docker Compose dentro de `monitoring-01`.

El proyecto se encuentra dentro de:

```text
~/docker/monitoring/
```

La estructura principal utilizada es:

```text
monitoring/
├── compose.yaml
└── prometheus/
    └── prometheus.yml
```

Los servicios principales son:

```text
Prometheus → puerto 9090
Grafana    → puerto 3000
```

Esto permite acceder desde mi PC a las interfaces web utilizando la dirección IP de `monitoring-01`.

Prometheus se encarga de recopilar y almacenar las métricas, mientras que Grafana se utiliza para consultarlas y representarlas gráficamente.

---

## Comunicación entre Grafana y Prometheus

Dentro de Grafana configuré Prometheus como fuente de datos.

La URL utilizada es:

```text
http://prometheus:9090
```

No se utiliza directamente la dirección IP de `monitoring-01`.

Esto es posible porque ambos contenedores forman parte del mismo proyecto Docker Compose y pueden comunicarse utilizando el nombre del servicio como nombre DNS.

De esta forma Grafana puede localizar el contenedor de Prometheus mediante el nombre:

```text
prometheus
```

Esta configuración también evita depender de la dirección IP interna que Docker asigne al contenedor.

---

## Node Exporter

Para obtener métricas del sistema operativo utilicé Node Exporter.

Node Exporter expone información del servidor a través del puerto:

```text
9100/tcp
```

Entre las métricas disponibles se encuentran datos relacionados con:

- CPU.
- Memoria.
- Sistemas de archivos.
- Interfaces de red.
- Carga del sistema.
- Tiempo de actividad.

Prometheus consulta periódicamente este endpoint y almacena las métricas obtenidas.

Actualmente los servidores que forman parte activa de la monitorización son:

```text
monitoring-01
srv-minecraft-01
```

También dejé definidos en Prometheus otros servidores del homelab que actualmente permanecen apagados y que no forman parte todavía de la monitorización activa.

---

## Configuración de Prometheus

Los servidores se definen dentro de:

```text
prometheus/prometheus.yml
```

Para facilitar su identificación añadí una etiqueta personalizada llamada `server`.

La configuración utilizada quedó de la siguiente forma:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets:
          - "192.168.0.3:9100"
        labels:
          server: "srv-linux-01"

      - targets:
          - "192.168.0.4:9100"
        labels:
          server: "docker-01"

      - targets:
          - "192.168.0.5:9100"
        labels:
          server: "monitoring-01"

      - targets:
          - "192.168.0.6:9100"
        labels:
          server: "srv-minecraft-01"
```

La etiqueta `server` permite trabajar en Grafana con nombres como:

```text
monitoring-01
srv-minecraft-01
```

en lugar de tener que identificar las máquinas mediante valores como:

```text
192.168.0.5:9100
192.168.0.6:9100
```

---

## Targets de Prometheus

Prometheus permite comprobar el estado de los servidores configurados como targets.

La métrica:

```promql
up{job="node_exporter"}
```

permite comprobar si Prometheus puede realizar correctamente el scraping de cada servidor.

Los valores posibles son:

```text
1 → Prometheus ha podido obtener las métricas.
0 → Prometheus no ha podido obtener las métricas.
```

Es importante tener en cuenta que un valor `1` no significa que todos los servicios del servidor funcionen correctamente.

Únicamente indica que Prometheus ha podido comunicarse con el endpoint que está monitorizando.

---

## Firewall y acceso a Node Exporter

Durante la configuración de `srv-minecraft-01`, Prometheus mostraba el servidor como `DOWN`.

Desde `monitoring-01` comprobé primero la conectividad mediante:

```bash
ping 192.168.0.6
```

La máquina respondía correctamente.

Después intenté acceder directamente a las métricas:

```bash
curl http://192.168.0.6:9100/metrics
```

La conexión no funcionaba.

En `srv-minecraft-01` comprobé que Node Exporter estaba activo y escuchando en el puerto 9100, por lo que el siguiente punto a revisar era el firewall.

El problema estaba en UFW.

En lugar de permitir el acceso al puerto 9100 desde toda la red, decidí limitarlo únicamente al servidor de monitorización:

```bash
sudo ufw allow from 192.168.0.5 to any port 9100 proto tcp
```

De esta forma:

```text
monitoring-01 → 192.168.0.5 → puede acceder a 9100
otros equipos                    → no necesitan acceso
```

Después de aplicar la regla, Prometheus pudo realizar correctamente el scraping y `srv-minecraft-01` pasó a aparecer como `UP`.

Esta configuración reduce la exposición innecesaria de Node Exporter dentro de la red.

---

## Dashboard de Grafana

En Grafana creé un dashboard llamado:

```text
Homelab - Servidores
```

El objetivo era evitar crear un dashboard independiente para cada máquina.

En su lugar, decidí crear un único dashboard reutilizable con un selector que permite elegir qué servidor quiero visualizar.

Los paneles principales son:

- RAM utilizada.
- CPU utilizada.
- Disco utilizado.
- Tráfico de red.
- Estado del servidor.

---

## Variable de servidor

Inicialmente las consultas utilizaban directamente la dirección IP de cada servidor.

Esto hacía que los paneles estuvieran ligados a una máquina concreta.

Para evitarlo creé una variable de Grafana llamada:

```text
server
```

La variable obtiene los valores de la etiqueta `server` definida previamente en Prometheus.

La configuración utiliza:

```text
Query type: Label values
Label: server
Metric: up
Filter: job = node_exporter
```

Esto permite seleccionar desde Grafana servidores como:

```text
monitoring-01
srv-minecraft-01
```

Los paneles utilizan posteriormente:

```promql
server="$server"
```

De esta forma el mismo conjunto de gráficas sirve para diferentes servidores sin tener que duplicar el dashboard.

---

## Uso de RAM

Para calcular el porcentaje de RAM utilizada utilizo:

```promql
100 * (1 - (
  node_memory_MemAvailable_bytes{server="$server"}
  /
  node_memory_MemTotal_bytes{server="$server"}
))
```

La consulta utiliza la memoria disponible en lugar de limitarse únicamente a la memoria marcada como libre.

Esto permite representar mejor la memoria que el sistema puede utilizar teniendo en cuenta el funcionamiento de la caché de Linux.

---

## Uso de CPU

Para calcular el uso global de CPU utilizo:

```promql
100 * (1 - avg by(server) (
  rate(node_cpu_seconds_total{
    server="$server",
    mode="idle"
  }[5m])
))
```

Node Exporter proporciona el tiempo empleado por cada CPU en diferentes estados.

En este caso se calcula el tiempo que la CPU no permanece en estado `idle` para obtener una aproximación del porcentaje total utilizado.

---

## Uso de disco

Para representar el porcentaje utilizado del sistema de archivos raíz utilizo:

```promql
100 * (
  1 -
  (
    node_filesystem_avail_bytes{
      server="$server",
      mountpoint="/",
      fstype!="rootfs"
    }
    /
    node_filesystem_size_bytes{
      server="$server",
      mountpoint="/",
      fstype!="rootfs"
    }
  )
)
```

Esto permite controlar el espacio disponible en el disco principal de cada servidor.

---

## Tráfico de red

Para visualizar el tráfico recibido utilizo:

```promql
rate(node_network_receive_bytes_total{
  server="$server",
  device="ens18"
}[5m])
```

Para el tráfico enviado:

```promql
rate(node_network_transmit_bytes_total{
  server="$server",
  device="ens18"
}[5m])
```

Grafana representa ambos valores como tasa de transferencia en bytes por segundo.

Esto permite observar la actividad de red de cada máquina a lo largo del tiempo.

---

## Estado del servidor

También añadí un panel para mostrar el estado del target seleccionado.

La consulta utilizada es:

```promql
up{job="node_exporter", server="$server"}
```

En Grafana configuré los siguientes valores:

```text
0 → DOWN
1 → UP
```

Este panel permite comprobar rápidamente si Prometheus está pudiendo obtener métricas del servidor.

---

## Problemas encontrados

### srv-minecraft-01 aparecía como DOWN

Uno de los primeros problemas apareció al añadir `srv-minecraft-01` a Prometheus.

El servidor respondía al ping, pero Prometheus no podía acceder al puerto 9100.

Para localizar el problema fui comprobando cada parte de la comunicación:

```text
monitoring-01
      │
      ▼
Conectividad IP
      │
      ▼
Puerto 9100
      │
      ▼
Firewall
      │
      ▼
Node Exporter
```

Node Exporter estaba activo y escuchando correctamente, por lo que finalmente el problema se localizó en UFW.

Después de permitir el puerto 9100 únicamente desde `192.168.0.5`, Prometheus pudo obtener las métricas correctamente.

Este problema me sirvió para comprobar la diferencia entre tener conectividad con una máquina y tener accesible un servicio concreto dentro de ella.

---

### Targets apagados

Prometheus también mostraba como `DOWN` algunos servidores configurados:

```text
srv-linux-01
docker-01
```

En este caso no existía ningún problema de red.

Las máquinas estaban apagadas intencionadamente y actualmente no necesito monitorizarlas.

Decidí mantenerlas definidas en `prometheus.yml` para tener preparada su configuración para cuando vuelvan a utilizarse.

Por este motivo, un target `DOWN` debe interpretarse teniendo en cuenta si realmente se espera que esa máquina esté disponible.

Esto será especialmente importante si en el futuro configuro alertas automáticas.

---

### Cambio de etiquetas en Prometheus

Inicialmente los targets no tenían la etiqueta personalizada `server`.

Después de añadirla, al realizar consultas en Prometheus aparecieron temporalmente series antiguas sin esta etiqueta y nuevas series que sí la contenían.

Esto no significaba que Prometheus estuviera monitorizando dos veces cada servidor.

Prometheus identifica las series temporales mediante el conjunto de etiquetas asociado a cada métrica. Al modificar las etiquetas se crean nuevas series, mientras que las anteriores permanecen almacenadas durante su periodo de retención.

La página de targets permitía comprobar que únicamente existían los targets configurados actualmente.

---

### No data en Grafana

Durante la creación de la variable de servidor algunos paneles mostraron:

```text
No data
```

Para diagnosticarlo comprobé primero las consultas directamente en Prometheus y posteriormente en Grafana Explore.

Una consulta directa como:

```promql
node_memory_MemTotal_bytes{server="srv-minecraft-01"}
```

permitió comprobar que la métrica y la etiqueta `server` existían correctamente.

También comprobé que las nuevas series solo tenían datos desde el momento en que se añadió la etiqueta.

Utilizar un intervalo reciente permitió verificar que Grafana estaba recibiendo correctamente los datos.

Este problema me sirvió para separar tres posibles puntos de fallo:

```text
Prometheus → métrica disponible
Grafana datasource → acceso a Prometheus
Panel → consulta y rango temporal
```

---

## Caso práctico: presión de memoria en Minecraft

Una de las primeras utilidades reales del sistema de monitorización apareció al observar el consumo de RAM de `srv-minecraft-01`.

Grafana mostraba un uso cercano al:

```text
93 %
```

En lugar de aumentar directamente la memoria de la máquina, comprobé primero el estado desde el propio servidor:

```bash
free -h
```

La máquina virtual tenía aproximadamente:

```text
RAM total:       3.8 GiB
RAM usada:       3.6 GiB
RAM disponible:  283 MiB
Swap utilizada:  362 MiB
```

El uso de swap y la poca memoria disponible indicaban que la máquina estaba funcionando con poco margen.

Para identificar el principal consumidor utilicé:

```bash
ps aux --sort=-%mem | head
```

El proceso Java utilizado por Minecraft estaba utilizando aproximadamente 3.3 GB de memoria física.

---

## Configuración de memoria de Java

El servidor utiliza Forge y los parámetros de memoria se encuentran en:

```text
/opt/minecraft-modded/data/user_jvm_args.txt
```

La configuración inicial era:

```text
-Xmx3G -Xms3G
```

Por tanto, Java podía utilizar un heap de 3 GB dentro de una máquina virtual que únicamente tenía 4 GB asignados.

Esto dejaba poco margen para:

- Debian.
- Docker.
- containerd.
- Node Exporter.
- Memoria utilizada por Java fuera del heap.
- Caché del sistema.

---

## Ampliación de memoria de la VM

Antes de modificar Java decidí aumentar primero la memoria asignada a la máquina virtual desde Proxmox.

La VM pasó de:

```text
4096 MiB
```

a:

```text
6144 MiB
```

Después de arrancar nuevamente el servidor mantuve inicialmente Java con el mismo límite de 3 GB para poder comparar el comportamiento.

Con Minecraft completamente iniciado, el sistema mostraba aproximadamente:

```text
RAM total:       5.8 GiB
RAM usada:       3.6 GiB
RAM disponible:  2.2 GiB
Swap utilizada:  0 B
```

Java seguía consumiendo aproximadamente la misma cantidad de memoria que antes.

Esto permitió comprobar que el problema principal no era un aumento repentino del consumo de Minecraft, sino que la máquina virtual estaba demasiado ajustada con 4 GB de RAM.

---

## Ajuste del heap de Java

Una vez comprobado que la VM disponía de margen suficiente, modifiqué los parámetros de Java.

La configuración pasó de:

```text
-Xms3G -Xmx3G
```

a:

```text
-Xms3G -Xmx4G
```

Con esta configuración Java comienza con un heap de 3 GB pero puede crecer hasta un máximo de 4 GB si la carga del servidor lo necesita.

La VM mantiene 6 GB de RAM, dejando margen para el sistema operativo y el resto de procesos.

Después de reiniciar el contenedor comprobé que Minecraft arrancaba correctamente y que el contenedor aparecía como:

```text
healthy
```

El sistema mostraba aproximadamente:

```text
RAM total:       5.8 GiB
RAM usada:       3.2 GiB
RAM disponible:  2.6 GiB
Swap utilizada:  0 B
```

A partir de este punto la idea es observar el comportamiento cuando haya varios jugadores conectados antes de realizar nuevas ampliaciones.

---

## Resultado

Al finalizar esta fase dispongo de una plataforma centralizada de monitorización formada por Prometheus, Grafana y Node Exporter.

Desde un único dashboard puedo seleccionar un servidor y consultar:

- Uso de CPU.
- Uso de RAM.
- Uso de disco.
- Tráfico de red.
- Estado del target.

La monitorización también permitió detectar un problema real de dimensionamiento en `srv-minecraft-01`.

En lugar de aumentar recursos únicamente por estimación, utilicé las métricas de Grafana junto con herramientas del propio sistema para comprobar el consumo real, identificar el proceso responsable y decidir la ampliación de memoria.

El servidor de Minecraft pasó de una VM con 4 GB de RAM y uso de swap a una VM con 6 GB, mayor margen de memoria disponible y Java configurado para poder utilizar hasta 4 GB de heap.

Esta fase me ha permitido utilizar la monitorización no solo para visualizar métricas, sino también como herramienta para diagnosticar problemas y tomar decisiones sobre los recursos del homelab.

---

## Próximos pasos

Los siguientes puntos que quiero añadir a la plataforma de monitorización son:

- Configurar alertas para detectar problemas automáticamente.
- Definir qué servidores se espera que estén disponibles permanentemente.
- Añadir nuevos servidores cuando vuelvan a ponerse en funcionamiento.
- Monitorizar servicios concretos además del estado general del sistema.
- Revisar umbrales de CPU, RAM y almacenamiento basándome en el uso real.

---

## Referencias

- Documentación oficial de Prometheus.
- Documentación oficial de Grafana.
- Documentación de Prometheus Node Exporter.
- Documentación oficial de Docker.