# 03 – Docker y administración con Portainer

## Objetivo

Documentar el despliegue y la administración de aplicaciones con Docker Compose y Portainer, incluyendo persistencia de datos, comprobación de servicios y diagnóstico.

Los ejemplos son una guía reproducible con valores genéricos. No representan la configuración exacta de la infraestructura personal.

---

## 1. Entorno de trabajo

Para este ejemplo se utiliza una VM Debian dedicada a las aplicaciones.

**Todos los comandos de este capítulo se ejecutan dentro de esa VM, no en el host Proxmox.**

Requisitos:

- Debian compatible con Docker Engine.
- Acceso administrativo mediante `sudo`.
- Conectividad y resolución DNS.
- Espacio suficiente para imágenes y datos.

Si se utiliza Docker dentro de LXC, la configuración requiere revisar compatibilidad, permisos y funciones del contenedor. Ese caso no se desarrolla en este procedimiento.

### Conceptos principales

| Elemento | Función |
|---|---|
| Imagen | Base utilizada para crear un contenedor |
| Contenedor | Instancia de una aplicación |
| Volumen | Almacenamiento persistente administrado por Docker |
| Bind mount | Carpeta o archivo del sistema accesible desde un contenedor |
| Red Docker | Comunicación entre contenedores |
| Docker Compose | Definición de servicios y recursos mediante YAML |
| Portainer | Interfaz web para administrar entornos Docker |

---

## 2. Instalar Docker Engine y Compose

Este procedimiento está destinado a una VM Debian nueva, sin instalaciones previas de Docker.

Si Docker ya está instalado, comprobar su funcionamiento antes de modificar paquetes o repositorios.

### Preparar el repositorio oficial

```bash
sudo apt update
sudo apt install ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Registrar el repositorio correspondiente a la versión de Debian:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

### Instalar los paquetes

```bash
sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

### Comprobar la instalación

```bash
sudo systemctl status docker --no-pager
sudo docker version
sudo docker compose version
sudo docker run --rm hello-world
```

La última comprobación descarga y ejecuta un contenedor de prueba, que se elimina al finalizar.

**Aprendizaje:** distinguir Docker Engine del plugin Compose y verificar que el motor puede ejecutar contenedores.

---

## 3. Preparar un proyecto de ejemplo

Se utiliza Portainer como aplicación de ejemplo para practicar Docker Compose y persistencia.

```bash
mkdir -p ~/docker/portainer
cd ~/docker/portainer
nano compose.yaml
```

Contenido de `compose.yaml`:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:lts
    restart: unless-stopped
    ports:
      - "127.0.0.1:9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

### Explicación de la configuración

| Parámetro | Función |
|---|---|
| `image` | Selecciona la imagen de Portainer CE |
| `restart` | Define la política de reinicio |
| `ports` | Publica la interfaz HTTPS únicamente en el propio servidor |
| `docker.sock` | Permite administrar el motor Docker local |
| `portainer_data` | Conserva la configuración de Portainer |

La etiqueta `lts` puede apuntar a distintas versiones con el tiempo. Para reproducir una versión exacta, registrar y fijar una etiqueta específica o un digest verificado.

> El acceso al socket de Docker otorga amplias capacidades de administración sobre la VM. Por eso el panel debe tener acceso restringido.

---

## 4. Validar y desplegar

**Ejecutar en:** la carpeta que contiene `compose.yaml`.

### Validar el archivo

```bash
sudo docker compose config --quiet
```

Si no muestra errores, la configuración supera la validación de Compose. Esto no garantiza todavía que la aplicación arranque correctamente.

### Descargar y ejecutar

```bash
sudo docker compose pull
sudo docker compose up -d
```

### Revisar el resultado

```bash
sudo docker compose ps
sudo docker compose logs --tail=100 portainer
```

**Qué se comprueba:** creación del contenedor, estado de ejecución y posibles errores de arranque.

Los registros pueden contener información sensible, como tokens de configuración inicial. Deben revisarse antes de publicar capturas.

---

## 5. Acceder a Portainer

En este ejemplo, el puerto se publica únicamente en `127.0.0.1`. Para acceder desde otro equipo se utiliza un túnel SSH.

**Ejecutar en:** el equipo desde el que se abrirá el navegador.

```bash
ssh -N -L 9443:127.0.0.1:9443 usuario@IP_DE_LA_VM
```

Mantener abierta la sesión y acceder a:

```text
https://localhost:9443
```

El certificado inicial puede ser autofirmado.

### Configuración inicial

1. Completar el asistente de configuración.
2. Introducir el token inicial si la versión instalada lo solicita.
3. Crear una cuenta administrativa con contraseña robusta.
4. Seleccionar el entorno Docker local.
5. Comprobar que aparecen sus contenedores, imágenes y volúmenes.

El túnel SSH evita publicar el panel en todas las interfaces de red.

---

## 6. Administración desde Portainer

### Contenedores

Permite consultar:

- Estado de ejecución.
- Imagen utilizada.
- Puertos publicados.
- Registros.
- Consumo de recursos.
- Volúmenes y redes asociados.

### Imágenes

Permite identificar las imágenes disponibles y las utilizadas por los contenedores.

### Volúmenes

Permite revisar los volúmenes de datos y su asociación con las aplicaciones.

### Stacks

Permite gestionar aplicaciones definidas mediante Compose desde la interfaz.

Para cada aplicación conviene mantener un método de gestión claro. Un proyecto desplegado desde consola puede aparecer en Portainer sin ofrecer todas las funciones de edición de un stack creado desde el panel.

**Aprendizaje:** utilizar la interfaz visual sin perder la relación entre contenedores, imágenes, redes y almacenamiento.

---

## 7. Persistencia de datos

Los datos escritos únicamente en la capa del contenedor pueden perderse al eliminarlo.

En el ejemplo, la configuración de Portainer se guarda en un volumen montado en `/data`.

### Consultar los volúmenes

```bash
sudo docker volume ls
```

Compose asigna normalmente un prefijo basado en el nombre del proyecto. Para conocer el nombre real se revisa la salida anterior.

### Comprobar los montajes del contenedor

Desde la carpeta del proyecto:

```bash
sudo docker inspect \
  "$(sudo docker compose ps -q portainer)" \
  --format '{{json .Mounts}}'
```

Esta comprobación permite identificar qué datos están almacenados fuera de la capa del contenedor.

> Un volumen persistente no es una copia de seguridad. También necesita una estrategia de respaldo.

---

## 8. Comandos habituales de administración

**Ejecutar en:** la carpeta del proyecto.

### Consultar el estado

```bash
sudo docker compose ps
```

### Consultar los registros

```bash
sudo docker compose logs --tail=100
```

### Seguir los registros en tiempo real

```bash
sudo docker compose logs --follow --tail=100
```

Para salir de la visualización, pulsar `Ctrl+C`. Esto no detiene la aplicación.

### Reiniciar un servicio

```bash
sudo docker compose restart portainer
```

Un reinicio no aplica todos los cambios realizados en `compose.yaml`. Para actualizar la definición del despliegue se utiliza `docker compose up -d`.

### Detener y volver a iniciar

```bash
sudo docker compose stop
sudo docker compose start
```

Estas operaciones conservan los contenedores existentes.

---

## 9. Diagnóstico

### Revisar recursos

```bash
sudo docker stats --no-stream
df -h
sudo docker system df
```

### Revisar el motor Docker

```bash
sudo systemctl status docker --no-pager
sudo journalctl -u docker -b --no-pager -n 100
```

### Problemas habituales

| Síntoma | Comprobaciones |
|---|---|
| No se puede descargar una imagen | Conectividad, DNS, nombre de imagen y etiqueta |
| El contenedor se reinicia repetidamente | Logs, configuración y permisos |
| La interfaz no responde | Estado del contenedor, puerto publicado y túnel SSH |
| Los datos no se conservan | Volúmenes y rutas montadas |
| Falta espacio | Uso de disco, imágenes y crecimiento de los datos |
| Compose no acepta el archivo | Sintaxis YAML, indentación y variables requeridas |

No se eliminan volúmenes como primera medida de diagnóstico: pueden contener datos necesarios.

---

## 10. Actualización y mantenimiento

Antes de actualizar una aplicación:

1. Revisar las notas de la versión.
2. Comprobar compatibilidad y requisitos.
3. Realizar una copia consistente de los datos.
4. Registrar la versión anterior.
5. Planificar la interrupción del servicio.

Después, desde la carpeta del proyecto:

```bash
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100
```

La validación incluye abrir la aplicación y comprobar que conserva su configuración.

En aplicaciones con bases de datos, volver a una imagen anterior puede no revertir cambios en los datos. La recuperación debe contemplar una copia compatible.

---

## 11. Validación final

- Docker Engine responde.
- Docker Compose reconoce la configuración.
- El contenedor permanece en ejecución.
- El panel es accesible mediante el método previsto.
- La cuenta administrativa está configurada.
- Los datos se guardan en un volumen.
- Los registros no muestran fallos que impidan el funcionamiento.
- El acceso administrativo está restringido.

---

## Conocimientos practicados

- Instalación y comprobación de Docker Engine.
- Definición de servicios con Docker Compose.
- Publicación controlada de puertos.
- Persistencia mediante volúmenes.
- Administración visual con Portainer.
- Consulta de registros y recursos.
- Diagnóstico y mantenimiento de aplicaciones.

---

## Referencias

- [Instalación de Docker Engine en Debian](https://docs.docker.com/engine/install/debian/)
- [Documentación de Docker Compose](https://docs.docker.com/compose/)
- [Instalación de Portainer CE en Docker](https://docs.portainer.io/start/install-ce/server/docker/linux)

---

[Volver al inicio](../README.md) · [Anterior: máquinas virtuales y LXC](02-maquinas-virtuales-lxc.md)
