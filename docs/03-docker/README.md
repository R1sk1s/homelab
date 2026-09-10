# Docker y gestión de contenedores

## Objetivo

El objetivo de esta máquina es disponer de un servidor dedicado donde poder desplegar diferentes servicios mediante Docker y acceder a ellos desde mi PC a través de la red local.

Además de desplegar servicios, quiero aprender cómo funciona Docker y entender conceptos como imágenes, contenedores, redes, publicación de puertos, volúmenes y Docker Compose.

Decidí utilizar una máquina virtual dedicada para Docker en lugar de instalarlo directamente sobre Proxmox, manteniendo así separado el hipervisor de los servicios que se ejecutarán en el homelab.

## Preparación de docker-01

Para crear `docker-01` decidí realizar un clon completo de `srv-linux-01` en lugar de instalar Debian nuevamente desde cero. De esta forma pude reutilizar la configuración base que ya tenía preparada y evitar repetir parte del trabajo.

Al clonar una máquina virtual también se copian elementos que identifican al sistema original, por lo que después del clonado fue necesario adaptar la nueva máquina.

Los principales cambios realizados fueron:

- Cambio del hostname a `docker-01`.
- Asignación de la IP estática `192.168.0.4/24`.
- Regeneración del `machine-id`.
- Regeneración de las claves SSH del servidor.

Después de realizar estos cambios comprobé que `srv-linux-01` y `docker-01` podían funcionar simultáneamente en la red sin conflictos.

## Instalación de Docker

Para instalar Docker decidí utilizar el repositorio oficial de Docker en lugar del paquete `docker.io` disponible directamente en los repositorios de Debian.

El objetivo era utilizar los paquetes mantenidos por Docker y disponer de Docker Engine junto con las herramientas y plugins oficiales, como Docker Compose y Buildx.

Para añadir el repositorio fue necesario importar su clave de firma y configurar APT para utilizar el repositorio correspondiente a Debian 13 (Trixie). Antes de realizar la instalación comprobé que el paquete `docker-ce` que iba a instalar procedía del repositorio oficial de Docker.

Los principales componentes instalados fueron:

- `docker-ce`: Docker Engine.
- `docker-ce-cli`: cliente de línea de comandos.
- `containerd.io`: runtime utilizado para gestionar el ciclo de vida de los contenedores.
- `docker-buildx-plugin`: extensión para la construcción de imágenes.
- `docker-compose-plugin`: permite gestionar aplicaciones definidas mediante Docker Compose.

Finalmente comprobé que el servicio de Docker estaba activo y que tanto Docker como Docker Compose estaban disponibles.

### Acceso a Docker sin sudo

Inicialmente mi usuario no tenía permisos para comunicarse con Docker mediante `/var/run/docker.sock`, por lo que `docker ps` fallaba mientras que `sudo docker ps` funcionaba.

El socket pertenece al grupo `docker`, así que añadí mi usuario a dicho grupo. Después de volver a iniciar la sesión SSH, pude utilizar Docker sin `sudo`.

Es importante tener en cuenta que pertenecer al grupo `docker` concede un nivel de acceso muy elevado sobre el sistema, por lo que este permiso debe limitarse a usuarios de confianza.

## Imágenes y contenedores

Una imagen Docker contiene la base necesaria para crear contenedores. A partir de una misma imagen se pueden crear múltiples contenedores independientes.

Durante las pruebas utilicé la imagen `nginx:latest` para crear varios contenedores Nginx. Esto me permitió comprobar que los cambios realizados directamente dentro de un contenedor no modifican la imagen original.

Al eliminar un contenedor y volver a crearlo desde la misma imagen, los cambios realizados dentro del contenedor se pierden si no se utiliza algún mecanismo de persistencia.

También comprobé la diferencia entre `docker run` y `docker start`: `docker run` crea un nuevo contenedor a partir de una imagen y lo inicia, mientras que `docker start` vuelve a iniciar un contenedor existente que se encuentra detenido.

## Publicación de puertos

Los contenedores pueden ejecutar servicios en sus propios puertos internos, pero esto no significa que esos servicios sean accesibles automáticamente desde otros equipos de la red.

Para hacer accesible el servicio Nginx desde mi PC publiqué el puerto 80 del contenedor mediante el puerto 8080 de `docker-01`:

```text
8080:80
```

El primer puerto corresponde al host y el segundo al contenedor. De esta forma, al acceder desde mi PC a `192.168.0.4:8080`, Docker redirige la conexión al puerto 80 del contenedor Nginx.

También creé un segundo contenedor Nginx sin publicar ningún puerto. Este servicio podía comunicarse con otros contenedores de su red Docker, pero no estaba publicado directamente en la red local.

## Redes Docker

Docker permite crear redes virtuales para controlar la comunicación entre los diferentes contenedores.

### Red bridge por defecto

Inicialmente utilicé la red `bridge` creada por Docker por defecto. Los contenedores recibieron direcciones IP dentro de la red privada de Docker y podían comunicarse directamente mediante esas direcciones.

Durante las pruebas comprobé que un contenedor podía acceder a otro utilizando su IP interna, pero la resolución por nombre no funcionaba como necesitaba.

### Red personalizada

Después creé una red bridge personalizada llamada `homelab-net` y conecté ambos contenedores a ella.

En una red bridge creada por el usuario, Docker proporciona resolución DNS interna entre los contenedores. Esto permitió que `nginx-web` pudiera comunicarse con `nginx-interno` utilizando su nombre:

```text
http://nginx-interno
```

De esta forma no es necesario depender de las direcciones IP internas de los contenedores, que pueden cambiar al recrearlos.

Las redes personalizadas también permiten separar grupos de contenedores y controlar qué servicios necesitan comunicarse entre sí.

## Persistencia de datos

Los cambios realizados directamente dentro de un contenedor pueden perderse cuando este se elimina y se crea uno nuevo desde la imagen.

Para comprobarlo modifiqué el contenido web de un contenedor Nginx y posteriormente lo eliminé y recreé. Al no utilizar almacenamiento persistente, el contenido volvió al estado original de la imagen.

### Volúmenes Docker

Después creé un volumen Docker llamado `nginx-data` y lo monté en:

```text
/usr/share/nginx/html
```

Volví a modificar el contenido web y repetí la prueba de eliminar y recrear el contenedor. Esta vez el contenido permaneció porque el volumen tiene un ciclo de vida independiente del contenedor.

Un volumen proporciona persistencia, pero no sustituye a una copia de seguridad, ya que los datos siguen dependiendo del almacenamiento del servidor.

## Docker Compose

Después de realizar las pruebas manualmente con `docker run`, utilicé Docker Compose para definir los servicios mediante un archivo `compose.yaml`.

Compose permite definir en un mismo archivo los servicios, imágenes, puertos, volúmenes y redes que necesita una aplicación. De esta forma no es necesario ejecutar y recordar varios comandos `docker run` para recrear el entorno.

### Creación del proyecto

Creé el proyecto dentro de:

```text
~/docker/nginx-lab/
```

El archivo `compose.yaml` utilizado durante el laboratorio quedó definido de la siguiente forma:

```yaml
services:
  nginx-web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - nginx-data:/usr/share/nginx/html
    networks:
      - homelab-net

  nginx-interno:
    image: nginx:latest
    networks:
      - homelab-net

volumes:
  nginx-data:

networks:
  homelab-net:
```

### Servicios

El proyecto contiene dos servicios:

- `nginx-web`: utiliza Nginx, publica el puerto 80 del contenedor mediante el puerto 8080 del host, utiliza almacenamiento persistente y está conectado a `homelab-net`.
- `nginx-interno`: utiliza la misma imagen Nginx y pertenece a `homelab-net`, pero no publica ningún puerto en el host.

Esto permite que `nginx-web` sea accesible desde mi PC mientras que `nginx-interno` se mantiene como un servicio interno dentro de la red Docker.

### Redes y volúmenes

Dentro de cada servicio se indica qué redes y volúmenes utiliza. Al final del archivo se declaran los recursos que serán gestionados por el proyecto Compose:

```yaml
volumes:
  nginx-data:

networks:
  homelab-net:
```

Al ejecutar:

```bash
docker compose up -d
```

Compose crea los recursos necesarios y levanta los servicios definidos en el archivo.

Al estar el proyecto dentro del directorio `nginx-lab`, Compose utiliza este nombre como nombre del proyecto y lo emplea como prefijo para los recursos que gestiona.

### Persistencia con Compose

Modifiqué el contenido web almacenado en el volumen gestionado por Compose y ejecuté:

```bash
docker compose down
```

Los contenedores y la red del proyecto fueron eliminados, pero el volumen permaneció.

Después ejecuté nuevamente:

```bash
docker compose up -d
```

Compose recreó los contenedores y la red, y el contenido web seguía disponible gracias al volumen persistente.

Esto también permite entender la diferencia entre el ciclo de vida de los contenedores y el de los datos persistentes.

## Problemas encontrados

### Machine ID duplicado después del clonado

Al crear `docker-01` mediante un clon de `srv-linux-01`, la nueva máquina heredó el mismo `machine-id` que el servidor original.

Inicialmente intenté regenerarlo eliminando `/etc/machine-id`, pero después de ejecutar `systemd-machine-id-setup` comprobé que el identificador seguía siendo el mismo.

El problema era que también existía `/var/lib/dbus/machine-id` con el identificador heredado del sistema original. Después de eliminar ambos archivos y volver a generar el `machine-id`, la nueva máquina obtuvo un identificador único.

Este problema me sirvió para comprobar la importancia de verificar el estado final de una configuración en lugar de asumir que un comando se ha aplicado correctamente solo porque no devuelve ningún error.

### Recursos creados por Docker Compose

Antes de utilizar Docker Compose ya había creado manualmente el volumen `nginx-data` y la red `homelab-net`.

Al ejecutar el proyecto con Docker Compose comprobé que se habían creado nuevos recursos llamados:

```text
nginx-lab_nginx-data
nginx-lab_homelab-net
```

Esto ocurre porque Compose utiliza el nombre del proyecto, en este caso `nginx-lab`, como prefijo para los recursos que gestiona.

Como consecuencia, el volumen utilizado por Compose no era el mismo volumen `nginx-data` que había creado anteriormente, por lo que el contenido persistente de las pruebas anteriores no apareció inicialmente.

Decidí continuar utilizando el volumen gestionado por Compose y comprobé nuevamente la persistencia eliminando y recreando los contenedores.

## Resultado

Al finalizar esta fase dispongo de una máquina virtual Debian dedicada a Docker dentro de Proxmox.

He desplegado servicios Nginx y realizado pruebas para entender el funcionamiento de imágenes, contenedores, publicación de puertos, redes bridge, DNS interno, volúmenes y Docker Compose.

El laboratorio actualmente permite desplegar varios servicios mediante un único archivo `compose.yaml`, mantener datos persistentes fuera del ciclo de vida de los contenedores y controlar qué servicios son accesibles desde la red local y cuáles permanecen únicamente dentro de la red Docker.

Esta máquina servirá como base para desplegar nuevos servicios del homelab en las siguientes fases.

## Referencias

- Documentación oficial de Docker.
- Documentación oficial de Docker Compose.
- Documentación de Debian.