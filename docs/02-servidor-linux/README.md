# Despliegue y configuración del primer servidor Linux

## Objetivo

El objetivo de esta máquina virtual es disponer del primer servidor Linux del homelab y utilizarlo como base para practicar la administración de sistemas en un entorno real.

Para ello desplegué una instalación de Debian orientada a servidor, sobre la que configuré aspectos como la red, resolución DNS, acceso remoto mediante SSH, seguridad y la integración de la máquina virtual con Proxmox.

Esta máquina también servirá como referencia para futuros servidores que vaya incorporando al laboratorio.

## Creación de la máquina virtual

Para desplegar el primer servidor decidí utilizar una máquina virtual completa en Proxmox en lugar de un contenedor LXC.

Aunque LXC permite ejecutar sistemas con un menor consumo de recursos, los contenedores comparten el kernel del host. En esta primera fase preferí utilizar una máquina virtual para disponer de un sistema Debian completo y con mayor aislamiento respecto al hipervisor.

La máquina virtual se creó inicialmente con los siguientes recursos:

| Recurso | Configuración |
|---|---|
| VMID | `100` |
| Hostname | `srv-linux-01` |
| CPU | 2 vCPU |
| RAM | 2 GiB |
| Disco | 32 GB |
| Controlador de disco | VirtIO SCSI |
| Red | VirtIO conectada a `vmbr0` |

Los recursos se han mantenido relativamente bajos porque el servidor no necesita una gran capacidad para las tareas que realiza actualmente. La idea es aumentar CPU, memoria o almacenamiento únicamente cuando el uso real del servidor lo justifique.

## Instalación de Debian

Como sistema operativo elegí Debian 13, realizando una instalación orientada a servidor y sin entorno gráfico.

Durante la instalación se habilitaron únicamente los componentes necesarios para disponer de un sistema base, incluyendo el servidor SSH y las utilidades estándar del sistema.

No instalé un entorno de escritorio porque la máquina está diseñada para administrarse remotamente. Esto permite reducir el consumo de recursos y evita instalar componentes gráficos que no son necesarios para la función del servidor.

Una vez finalizada la instalación, la administración del servidor se realiza principalmente mediante SSH desde mi equipo.

## Configuración de red

Durante la instalación inicial, `srv-linux-01` obtuvo automáticamente mediante DHCP la dirección `192.168.0.36`.

Al tratarse de un servidor decidí sustituir esta configuración por una dirección IP estática. Esto permite que otros equipos y servicios del homelab puedan localizar siempre el servidor en la misma dirección.

La configuración utilizada es:

| Parámetro | Valor |
| Interfaz | `ens18` |
| Dirección IP | `192.168.0.3/24` |
| Gateway | `192.168.0.1` |
| DNS | `1.1.1.1` |

La dirección `192.168.0.3` se encuentra fuera del rango DHCP utilizado por el router (`192.168.0.10 - 192.168.0.250`), evitando que el servidor DHCP pueda asignar la misma dirección a otro dispositivo.

Debian utiliza `ifupdown` en esta instalación, por lo que la configuración persistente de la interfaz se realizó en `/etc/network/interfaces`.

## Configuración de DNS

Después de configurar la dirección IP estática en `srv-linux-01`, apareció un problema con la resolución de nombres.

Al realizar las primeras comprobaciones observé que el servidor podía comunicarse correctamente con direcciones IP externas utilizando `ping 1.1.1.1`.

Sin embargo, al intentar hacer `ping debian.org`, el servidor devolvía el error `Temporary failure in name resolution`.

### Diagnóstico

El hecho de poder comunicarme con `1.1.1.1` indicaba que el servidor tenía conectividad con Internet y que la configuración básica de red estaba funcionando.

El problema aparecía únicamente cuando utilizaba nombres de dominio, por lo que centré el diagnóstico en la configuración DNS.

Al revisar `/etc/resolv.conf` comprobé que no había ningún servidor DNS configurado.

Antes de configurar una dirección IP estática, la máquina obtenía su configuración de red mediante DHCP. Al dejar de utilizar DHCP, también dejó de recibir automáticamente la configuración DNS.

### Solución

Como primera prueba configuré manualmente el servidor DNS `1.1.1.1` en `/etc/resolv.conf`.

Después de realizar este cambio, `srv-linux-01` volvió a resolver correctamente nombres como `debian.org`, confirmando que el origen del problema era la configuración DNS.

Esta modificación servía para comprobar el diagnóstico, pero no quería depender de una configuración manual de `/etc/resolv.conf`.

Para hacer persistente la configuración instalé `resolvconf` y añadí el servidor DNS a `/etc/network/interfaces`.

La configuración final de `ens18` quedó de la siguiente forma:

    allow-hotplug ens18
    iface ens18 inet static
        address 192.168.0.3/24
        gateway 192.168.0.1
        dns-nameservers 1.1.1.1

### Verificación

Después de reiniciar el servidor comprobé que `/etc/resolv.conf` se generaba correctamente e incluía:

    nameserver 1.1.1.1

Finalmente realicé nuevamente las pruebas de conectividad.

La comunicación mediante dirección IP funcionaba correctamente y también se podían resolver nombres de dominio, confirmando que la configuración DNS permanecía después de reiniciar la máquina.

Esta incidencia me permitió diferenciar entre un problema de conectividad y un problema de resolución DNS: si existe comunicación con una IP externa pero falla el acceso mediante nombres de dominio, la resolución DNS es uno de los primeros elementos que se deben comprobar.

## Acceso mediante SSH

Durante la instalación de Debian se instaló el servidor SSH para poder administrar `srv-linux-01` de forma remota.

Al tratarse de un servidor sin entorno gráfico, SSH se convierte en el método principal de administración. Esto permite trabajar directamente desde mi equipo Windows sin necesidad de utilizar constantemente la consola de Proxmox.

Una vez configurada la red y comprobado que el servicio SSH estaba activo, pude conectarme inicialmente utilizando:

    ssh bryan@192.168.0.3

### Configuración del cliente SSH en Windows

Para simplificar el acceso a los servidores del homelab configuré el archivo `~/.ssh/config` del cliente SSH de mi equipo.

Para `srv-linux-01` añadí la siguiente configuración:

    Host srv-linux-01
        HostName 192.168.0.3
        User bryan
        IdentityFile ~/.ssh/id_ed25519_homelab
        IdentitiesOnly yes

De esta forma no es necesario recordar y escribir en cada conexión la dirección IP, el usuario y la clave SSH correspondiente.

El acceso al servidor puede realizarse simplemente mediante:

    ssh srv-linux-01

Esta configuración también permitirá ir añadiendo el resto de servidores del homelab al mismo archivo y administrarlos utilizando sus nombres.

## Hardening de SSH

Una vez comprobado que el acceso mediante SSH funcionaba correctamente, decidí mejorar la seguridad del servicio sustituyendo la autenticación mediante contraseña por autenticación mediante claves SSH.

### Autenticación mediante claves

Desde mi equipo Windows generé un par de claves ED25519 específico para el homelab.

El par está formado por:

- **Clave privada:** permanece únicamente en mi equipo y no debe compartirse.
- **Clave pública:** puede almacenarse en los servidores a los que quiero tener acceso.

La clave privada permanece únicamente en mi equipo de administración y no se comparte con los servidores.

La clave pública correspondiente se añadió en el servidor al archivo:

    ~/.ssh/authorized_keys

También se configuraron los permisos correspondientes:

    ~/.ssh                  700
    ~/.ssh/authorized_keys  600

Después de añadir la clave pública comprobé que podía iniciar sesión correctamente utilizando la clave SSH antes de realizar cambios adicionales en la configuración del servidor.

### Desactivación de autenticación mediante contraseña

Una vez verificado el acceso mediante clave, desactivé la autenticación SSH mediante contraseña.

En lugar de modificar directamente toda la configuración principal de OpenSSH, creé el archivo:

    /etc/ssh/sshd_config.d/10-hardening.conf

con la siguiente configuración:

    PubkeyAuthentication yes
    PasswordAuthentication no
    KbdInteractiveAuthentication no

De esta forma se permite la autenticación mediante clave pública y se deshabilitan los métodos de autenticación mediante contraseña utilizados para el acceso SSH.

Antes de aplicar los cambios comprobé que la configuración no contenía errores de sintaxis mediante:

    sudo sshd -t

El comando no devolvió ningún error, indicando que la configuración era válida.

También comprobé la configuración efectiva del servidor SSH mediante:

    sudo sshd -T

Verificando que los valores aplicados eran:

    pubkeyauthentication yes
    passwordauthentication no
    kbdinteractiveauthentication no

Después de estas comprobaciones recargué el servicio SSH para aplicar la nueva configuración.

### Verificación

Para asegurarme de que la autenticación mediante contraseña estaba realmente deshabilitada realicé una conexión forzando al cliente SSH a no utilizar la clave pública:

    ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password bryan@192.168.0.3

El servidor rechazó la conexión mostrando:

    Permission denied (publickey)

Finalmente comprobé que el acceso normal mediante la clave SSH continuaba funcionando correctamente:

    ssh srv-linux-01

De esta forma, el acceso remoto a `srv-linux-01` queda configurado mediante autenticación por clave pública, mientras que el acceso SSH mediante contraseña permanece deshabilitado.

## QEMU Guest Agent

Para mejorar la integración entre Proxmox y `srv-linux-01` configuré QEMU Guest Agent dentro de la máquina virtual.

La máquina virtual puede funcionar sin este agente, pero Proxmox tendría menos información y capacidad de comunicación con el sistema operativo que se está ejecutando dentro de ella.

QEMU Guest Agent actúa como un canal de comunicación entre el hipervisor y el sistema operativo invitado.

### Instalación y configuración

En Debian comprobé que el paquete `qemu-guest-agent` estaba instalado y que el servicio se encontraba activo:

    systemctl status qemu-guest-agent

Además, habilité la opción **QEMU Guest Agent** desde la configuración de la máquina virtual en Proxmox.

De esta forma, Proxmox puede comunicarse con el agente que se ejecuta dentro de Debian.

### Verificación

Desde el nodo Proxmox comprobé que existía comunicación con el agente mediante:

    qm agent 100 ping

El comando se ejecutó correctamente, confirmando que Proxmox podía comunicarse con QEMU Guest Agent dentro de la VM.

También comprobé que Proxmox podía obtener información de las interfaces de red del sistema invitado:

    qm guest cmd 100 network-get-interfaces

Entre la información obtenida aparecía la interfaz `ens18` con la dirección IP:

    192.168.0.3

Esto confirma que el hipervisor puede obtener información directamente desde el sistema operativo invitado a través del agente.

### Integración con snapshots

QEMU Guest Agent también puede colaborar con Proxmox durante determinadas operaciones sobre la máquina virtual.

Al crear un snapshot observé las siguientes operaciones:

    freeze guest filesystem
    snapshot
    thaw guest filesystem

Antes de realizar el snapshot, Proxmox solicita al agente congelar temporalmente las operaciones del sistema de archivos. Una vez creado el snapshot, el sistema de archivos vuelve a su funcionamiento normal.

Esto ayuda a conseguir un estado más consistente de los datos durante la creación del snapshot, especialmente cuando la máquina virtual se encuentra encendida.

## Snapshots

Una vez terminada la configuración inicial de `srv-linux-01`, utilicé los snapshots de Proxmox para guardar estados concretos de la máquina virtual antes de continuar realizando cambios.

El objetivo de estos snapshots es disponer de puntos a los que poder volver rápidamente si una configuración posterior provoca algún problema.

### Snapshot inicial

Después de completar la instalación de Debian y configurar correctamente la red, SSH y QEMU Guest Agent, creé el primer snapshot:

    baseline-debian

Este snapshot representa un estado base funcional del servidor antes de comenzar a realizar configuraciones adicionales.

La descripción utilizada fue:

    Debian 13 baseline: static network, ssh and QEMU Guest Agent configured.

No incluí el estado de la memoria RAM, ya que el objetivo era guardar el estado de los discos de la máquina virtual y no restaurar una sesión concreta que estuviese ejecutándose en ese momento.

### Snapshot después del hardening de SSH

Después de configurar la autenticación mediante claves y deshabilitar el acceso SSH mediante contraseña, creé un segundo snapshot:

    ssh-hardened

Este snapshot representa un nuevo punto estable después de aplicar las medidas de seguridad sobre SSH.

De esta forma puedo distinguir entre diferentes estados de configuración de la máquina:

    baseline-debian
           |
           | Configuración y hardening de SSH
           v
      ssh-hardened
           |
           | Futuros cambios
           v
          ...

Esta forma de trabajar permite crear puntos de recuperación antes de realizar cambios importantes en el servidor.

### Snapshot y backup

Es importante diferenciar un snapshot de una copia de seguridad.

En mi infraestructura actual, tanto la máquina virtual como sus snapshots se encuentran almacenados en el mismo SSD NVMe del servidor Proxmox.

Por este motivo, los snapshots son útiles para recuperar rápidamente un estado anterior después de un error de configuración, pero no protegen frente a problemas como el fallo físico del SSD o la pérdida completa del servidor.

Por ejemplo, si una configuración incorrecta provoca que el servidor deje de funcionar correctamente, podría utilizar un snapshot para volver a un estado anterior.

Sin embargo, si falla el SSD donde se encuentran almacenados Proxmox, la máquina virtual y sus snapshots, perdería todos esos datos.

Por este motivo utilizaré los snapshots como puntos de recuperación durante la configuración del laboratorio, pero más adelante será necesario implementar una estrategia de backups almacenados en una ubicación diferente.

## Problemas encontrados

Durante la configuración de `srv-linux-01` aparecieron varios problemas que fue necesario diagnosticar y solucionar.

### Dirección DHCP anterior después de configurar la IP estática

Inicialmente el servidor obtenía mediante DHCP la dirección:

    192.168.0.36

Posteriormente configuré manualmente la dirección estática:

    192.168.0.3/24

Después de modificar `/etc/network/interfaces` y reiniciar el servicio de red, comprobé que la interfaz `ens18` seguía teniendo las dos direcciones IP asignadas.

Esto me permitió ver la diferencia entre la configuración persistente almacenada en los archivos del sistema y el estado de red que se encuentra activo en ese momento.

Aunque `/etc/network/interfaces` ya contenía la nueva configuración estática, la dirección obtenida anteriormente mediante DHCP todavía permanecía activa en el sistema.

Después de reiniciar la máquina virtual, la interfaz arrancó utilizando únicamente la configuración persistente:

    192.168.0.3/24

También comprobé que la ruta por defecto utilizaba correctamente:

    192.168.0.1

### Problema de resolución DNS

Después de configurar la dirección IP estática, el servidor mantenía conectividad mediante IP pero no podía resolver nombres de dominio.

Las pruebas con `ping 1.1.1.1` y `ping debian.org` permitieron aislar el problema en la resolución DNS.

Tras revisar `/etc/resolv.conf`, configuré primero un servidor DNS de forma temporal para confirmar el diagnóstico y posteriormente hice persistente la configuración mediante `resolvconf`.

El proceso completo se encuentra documentado en el apartado [Configuración de DNS](#configuración-de-dns).

### Archivo de configuración SSH en Windows

Al configurar un alias para conectarme al servidor mediante:

    ssh srv-linux-01

el cliente SSH de Windows no reconocía inicialmente el nombre configurado.

Al revisar el contenido del directorio `.ssh` descubrí que el archivo se había guardado como:

    config.txt

en lugar de:

    config

OpenSSH busca por defecto un archivo llamado `config`, sin extensión, por lo que la configuración no estaba siendo cargada.

Después de renombrar el archivo, el alias comenzó a funcionar correctamente.

### Aprendizajes

Estos problemas me permitieron practicar un proceso de diagnóstico basado en comprobar cada parte del sistema por separado antes de realizar cambios.

En lugar de limitarme a aplicar una solución, intenté identificar primero qué componente estaba fallando y verificar después que el cambio realizado solucionaba realmente el problema.

## Resultado

Después de completar esta fase, `srv-linux-01` queda configurado como el primer servidor Linux del homelab y preparado para ser administrado de forma remota.

El estado actual del servidor es:

| Elemento | Configuración |
|---|---|
| Sistema operativo | Debian 13 |
| Hostname | `srv-linux-01` |
| Dirección IP | `192.168.0.3/24` |
| Gateway | `192.168.0.1` |
| DNS | `1.1.1.1` |
| CPU | 2 vCPU |
| RAM | 2 GiB |
| Disco | 32 GB |
| Administración remota | SSH |
| Autenticación SSH | Clave ED25519 |
| Contraseña mediante SSH | Deshabilitada |
| QEMU Guest Agent | Activo |
| Snapshot base | `baseline-debian` |
| Snapshot tras hardening | `ssh-hardened` |

Esta máquina me ha servido para preparar una base de servidor Debian sobre Proxmox y trabajar aspectos de administración como configuración de red, DNS, acceso remoto, autenticación mediante claves y comunicación entre el sistema invitado y el hipervisor.

También aparecieron diferentes problemas durante la configuración que permitieron practicar el diagnóstico de red y servicios en lugar de limitar el proceso únicamente a la instalación del sistema.

`srv-linux-01` queda como servidor de referencia dentro del laboratorio y como base para seguir incorporando nuevos servicios y máquinas a la infraestructura.

## Referencias

Para realizar y documentar esta parte del proyecto se ha utilizado principalmente documentación oficial de las tecnologías empleadas:

- Debian Administrator's Handbook
- Debian Reference
- OpenSSH Manual Pages
- Proxmox VE Documentation
- QEMU Guest Agent Documentation