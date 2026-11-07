# homelab-proxmox

# Laboratorio de virtualización y administración de sistemas

Proyecto práctico de administración de sistemas con **Proxmox VE, Linux, máquinas virtuales, contenedores LXC y Docker**.

Esta documentación recoge los conocimientos adquiridos y los procedimientos principales utilizados para desplegar, administrar y mantener servicios autohospedados.

![Proxmox](https://img.shields.io/badge/Proxmox-E57000?style=for-the-badge&logo=proxmox&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Portainer](https://img.shields.io/badge/Portainer-13BEF9?style=for-the-badge&logo=portainer&logoColor=white)


---

## 1. Conocimientos adquiridos

| Área | Prácticas y aprendizaje |
|---|---|
| Virtualización | Administración de máquinas virtuales y contenedores LXC con Proxmox VE |
| Linux | Comprobación de servicios, recursos, almacenamiento y registros |
| Contenedores | Gestión de aplicaciones mediante Docker y Docker Compose |
| Almacenamiento | Revisión de discos, sistemas de archivos y puntos de montaje |
| Monitorización | Comprobación de disponibilidad con Uptime Kuma |
| Administración | Gestión de entornos Docker mediante Portainer |
| Backups | Configuración y revisión de copias de seguridad en Proxmox |
| Diagnóstico | Identificación de errores a partir del estado de los servicios y sus logs |

---

## 2. Preparación de Proxmox VE

La instalación inicial se realiza desde la imagen ISO de Proxmox VE.

### Procedimiento general

1. Preparar un USB de instalación con la ISO oficial.
2. Arrancar el equipo desde el USB.
3. Seleccionar el disco destinado a Proxmox.
4. Configurar las credenciales administrativas.
5. Establecer los parámetros de red y el nombre del equipo.
6. Finalizar la instalación y acceder a la interfaz web.

> El disco seleccionado durante la instalación se formatea. Es necesario comprobar el destino antes de continuar.

Acceso a la interfaz:

```text
https://IP_DEL_HOST:8006
```

### Comprobar la instalación

**Ejecutar en:** consola del host Proxmox.

```bash
# Consultar la versión instalada
pveversion

# Revisar el estado de los servicios principales
systemctl status pveproxy pvedaemon pvestatd --no-pager

# Consultar los almacenamientos configurados
pvesm status
```

**Qué se comprueba:** versión de la plataforma, funcionamiento de sus servicios y disponibilidad del almacenamiento.

---

## 3. Administración de máquinas virtuales

Las máquinas virtuales permiten ejecutar un sistema operativo con su propio kernel y recursos asignados.

### Creación desde la interfaz web

1. Subir la ISO del sistema operativo a un almacenamiento que admita imágenes ISO.
2. Seleccionar **Create VM**.
3. Asignar un identificador y un nombre.
4. Seleccionar la ISO.
5. Configurar disco, CPU, memoria y red.
6. Revisar la configuración e iniciar la instalación.

Los recursos se ajustan según las necesidades del sistema invitado y la capacidad disponible.

### Consultar el estado desde consola

**Ejecutar en:** host Proxmox.

En este ejemplo, `200` representa el identificador de una VM de laboratorio ya creada.

```bash
# Listar las máquinas virtuales
qm list

# Consultar su configuración
qm config 200

# Comprobar su estado
qm status 200
```

### Gestionar el encendido y apagado

```bash
# Iniciar la máquina virtual
qm start 200

# Solicitar un apagado ordenado
qm shutdown 200
```

**Aprendizaje:** administrar el ciclo de vida de una VM y comprobar su configuración tanto desde la interfaz gráfica como desde la consola.

---

## 4. Administración de contenedores LXC

Los contenedores LXC permiten ejecutar entornos Linux que comparten el kernel del host.

### Creación desde la interfaz web

1. Descargar una plantilla desde un almacenamiento que admita plantillas de contenedores.
2. Seleccionar **Create CT**.
3. Asignar un identificador y un nombre.
4. Configurar el acceso administrativo.
5. Seleccionar la plantilla.
6. Asignar almacenamiento, CPU y memoria.
7. Configurar red y DNS.
8. Revisar las opciones e iniciar el contenedor.

Para un servicio Linux compatible, se prioriza un contenedor **sin privilegios**, salvo que exista una necesidad técnica justificada.

### Consultar y administrar el contenedor

**Ejecutar en:** host Proxmox.

En este ejemplo, `201` representa un contenedor de laboratorio ya creado.

```bash
# Listar los contenedores
pct list

# Consultar su configuración
pct config 201

# Comprobar su estado
pct status 201

# Iniciar el contenedor
pct start 201

# Acceder a su consola
pct enter 201
```

Para salir de la consola del contenedor:

```bash
exit
```

También es posible ejecutar una comprobación sin abrir una sesión interactiva:

```bash
pct exec 201 -- df -h
```

**Aprendizaje:** distinguir los comandos del host de los ejecutados dentro del contenedor y administrar cada entorno de forma independiente.

---

## 5. Comprobaciones básicas en Linux

Antes de desplegar o diagnosticar un servicio, se revisan los recursos y el estado del sistema.

**Ejecutar en:** VM o contenedor Linux que se esté administrando.

```bash
# Identificar el sistema operativo
cat /etc/os-release

# Consultar la memoria disponible
free -h

# Revisar el espacio de los sistemas de archivos
df -h

# Consultar las interfaces y direcciones de red
ip -br address

# Revisar las rutas de red
ip route

# Consultar los puertos en escucha
ss -tuln

# Detectar servicios fallidos
systemctl --failed
```

En sistemas con systemd, los registros permiten ampliar el diagnóstico:

```bash
# Consultar avisos y errores del arranque actual
journalctl -b -p warning --no-pager -n 100
```

**Qué se comprueba:** falta de memoria, almacenamiento lleno, errores de red y servicios que no han arrancado correctamente.

---

## 6. Gestión de aplicaciones con Docker Compose

Docker Compose permite describir una aplicación y sus servicios en un archivo YAML.

En las prácticas se ha trabajado con despliegues de aplicaciones autohospedadas, persistencia de datos y revisión de registros.

> Los siguientes comandos requieren Docker Engine y el plugin Docker Compose instalados dentro del entorno destinado a las aplicaciones.

**Ejecutar en:** VM o contenedor que aloja Docker, dentro de la carpeta de la aplicación. No en el host Proxmox.

### 6.1. Comprobar las herramientas

```bash
docker --version
docker compose version
```

### 6.2. Validar la configuración

Desde una carpeta que contenga un archivo `compose.yaml` o `docker-compose.yml`:

```bash
docker compose config --quiet
```

Comprueba la configuración sin imprimir el contenido completo, que podría incluir variables sensibles.

### 6.3. Desplegar los servicios

Una vez revisadas las imágenes, los volúmenes y las variables:

```bash
# Descargar las imágenes definidas
docker compose pull

# Crear e iniciar los servicios
docker compose up -d

# Comprobar el estado
docker compose ps
```

### 6.4. Consultar registros y recursos

```bash
# Revisar los últimos registros
docker compose logs --tail=100

# Consultar el consumo de recursos
docker stats --no-stream
```

Que un contenedor aparezca como iniciado no garantiza que la aplicación funcione correctamente. También se comprueba el acceso al servicio y, cuando existe, su estado de salud.

**Aprendizaje:** interpretar archivos Compose, distinguir imágenes de contenedores y mantener los datos mediante volúmenes o montajes persistentes.

---

## 7. Monitorización y diagnóstico de servicios

Se ha trabajado con **Uptime Kuma** para monitorizar disponibilidad y con **Portainer** para administrar entornos Docker.

### Procedimiento de monitorización

1. Seleccionar el servicio que se quiere comprobar.
2. Elegir el tipo de monitor: HTTP, TCP o ping, según el caso.
3. Configurar el destino y el intervalo.
4. Revisar el estado y el historial de respuestas.
5. Investigar los fallos mediante comprobaciones de red y registros.

### Ejemplo de comprobación HTTP

**Ejecutar en:** un equipo que deba tener acceso al servicio.

```bash
curl --output /dev/null \
  --silent \
  --show-error \
  --max-time 10 \
  --write-out 'HTTP: %{http_code}\nTiempo: %{time_total}s\n' \
  https://servicio.example.com
```

El dominio es ilustrativo y debe sustituirse por el destino de laboratorio.

### Secuencia de diagnóstico

| Comprobación | Qué permite investigar |
|---|---|
| Estado de la VM o LXC | Si el entorno está encendido |
| Memoria y espacio libre | Si existe agotamiento de recursos |
| Puertos en escucha | Si el servicio está aceptando conexiones |
| Respuesta HTTP | Si la aplicación responde desde el punto de prueba |
| Logs de la aplicación | Qué errores registra el servicio |

**Aprendizaje:** relacionar una alerta de disponibilidad con comprobaciones técnicas para localizar su causa.

---

## 8. Copias de seguridad en Proxmox

Se han configurado y revisado tareas de backup de entornos virtualizados.

### Configuración desde la interfaz

1. Acceder a **Datacenter → Backup**.
2. Crear una tarea.
3. Seleccionar un almacenamiento que admita backups.
4. Elegir las VMs y los contenedores incluidos.
5. Definir el horario y la retención.
6. Seleccionar el modo de copia adecuado.
7. Revisar el registro al terminar.

### Ejemplo de copia manual

**Ejecutar en:** host Proxmox.

El identificador `201` y el almacenamiento `backup-lab` son ejemplos. El almacenamiento debe estar configurado previamente y admitir copias de seguridad.

```bash
vzdump 201 \
  --storage backup-lab \
  --mode snapshot \
  --compress zstd
```

La disponibilidad del modo snapshot depende del tipo de entorno y de su almacenamiento.

### Revisar el contenido del destino

```bash
pvesm list backup-lab --content backup
```

Además de comprobar que existe el archivo, se revisa que la tarea haya finalizado correctamente.

### Aspectos aprendidos

- Diferencia entre una instantánea y una copia de seguridad.
- Selección de los entornos incluidos en una tarea.
- Revisión de errores y consumo de almacenamiento.
- Importancia de definir una política de retención.
- Necesidad de probar la restauración.

> En LXC es necesario revisar los puntos de montaje: no todos sus datos se incluyen automáticamente en el backup. Los montajes bind y de dispositivos requieren una estrategia de copia específica.

### Restauración

La prueba de recuperación debe realizarse con un identificador nuevo y una red aislada para evitar conflictos con el servicio original.

La validación debe comprobar:

- Arranque del entorno recuperado.
- Integridad y disponibilidad de los datos.
- Funcionamiento de la aplicación.
- Tiempo necesario para recuperar el servicio.


---

## 9. Aplicación práctica de lo aprendido

| Caso trabajado | Experiencia adquirida |
|---|---|
| Home Assistant OS | Administración de una aplicación desplegada como máquina virtual |
| Servicios Zigbee/MQTT | Gestión de servicios Linux y comunicación entre aplicaciones |
| Immich | Despliegue de una aplicación, migración de fotografías y revisión del almacenamiento |
| Portainer | Administración visual de contenedores Docker |
| Uptime Kuma | Monitorización de disponibilidad e investigación de interrupciones |
| Backups de Proxmox | Programación de copias y revisión de los resultados |


---

## Referencias

- [Documentación de Proxmox VE](https://pve.proxmox.com/pve-docs/)
- [Documentación de Docker Compose](https://docs.docker.com/compose/)
- [Proyecto Nextcloud en Raspberry Pi 5](https://github.com/luzu1/rpi5-nextcloud)
