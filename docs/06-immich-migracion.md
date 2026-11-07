# 06 – Immich y migración de fotografías

## Objetivo

Documentar el despliegue de un servicio de gestión de fotografías y vídeos, la preparación de una migración y las comprobaciones del contenido importado.

El trabajo realizado incluye el uso de Immich, la migración de contenido desde Google Photos y la revisión del almacenamiento y las copias del contenedor.

Los comandos siguientes son ejemplos genéricos para una instalación de laboratorio. No representan un registro literal del despliegue original.

---

## 1. Componentes del servicio

| Componente | Función |
|---|---|
| Aplicación Immich | Acceso y gestión de fotografías y vídeos |
| PostgreSQL | Información de usuarios, archivos y relaciones |
| Procesamiento multimedia | Generación de miniaturas y otras tareas |
| Almacenamiento multimedia | Conservación de los archivos |
| Docker Compose | Definición y administración del despliegue |

La base de datos y los archivos multimedia deben considerarse conjuntamente al preparar una recuperación.

---

## 2. Preparar el entorno

**Ejecutar en:** el sistema Linux destinado a Docker, no en el host Proxmox.

Comprobar las herramientas y los recursos:

```bash
sudo docker version
sudo docker compose version
free -h
df -h
```

Antes de importar una colección, prever espacio para:

- Archivos originales.
- Miniaturas y contenido generado.
- Base de datos.
- Exportaciones temporales.
- Copias de seguridad.

El tamaño de la exportación no equivale al espacio total que necesitará el servicio.

---

## 3. Obtener los archivos de despliegue

Este procedimiento es para una instalación nueva. No descargar estos archivos encima de una configuración existente.

```bash
mkdir -p ~/docker/immich-lab
cd ~/docker/immich-lab
```

Descargar los archivos publicados por Immich:

```bash
curl -fL \
  https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml \
  -o docker-compose.yml

curl -fL \
  https://github.com/immich-app/immich/releases/latest/download/example.env \
  -o .env
```

Revisarlos antes de iniciar el servicio:

```bash
less docker-compose.yml
nano .env
```

Los enlaces apuntan a la última publicación. Registrar la versión elegida y conservar los archivos correspondientes para poder reproducir el despliegue.

---

## 4. Configurar almacenamiento y variables

En `.env`, revisar al menos:

| Variable | Qué configurar |
|---|---|
| `UPLOAD_LOCATION` | Directorio destinado a los archivos multimedia |
| `DB_DATA_LOCATION` | Directorio local de datos de PostgreSQL |
| `IMMICH_VERSION` | Versión compatible con los archivos descargados |
| `DB_PASSWORD` | Contraseña robusta y única |
| `TZ` | Zona horaria |

Mantener las demás variables necesarias del archivo oficial.

La ubicación de PostgreSQL debe utilizar almacenamiento local compatible, no una carpeta de red.

Proteger el archivo:

```bash
chmod 600 .env
```

No publicar `.env` en el repositorio. Si se documentan las variables, utilizar un archivo de ejemplo sin credenciales.

### Acceso al servicio

Revisar la sección `ports` del servicio web.

Para un laboratorio accesible mediante túnel SSH, sustituir la publicación del puerto web por:

```yaml
ports:
  - "127.0.0.1:2283:2283"
```

Modificar únicamente ese apartado del archivo oficial, conservando los servicios y dependencias restantes.

---

## 5. Desplegar y comprobar

Desde la carpeta del proyecto:

```bash
sudo docker compose config --quiet
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
```

Consultar registros:

```bash
sudo docker compose logs --tail=100
```

Revisar errores de conexión a la base de datos, permisos o espacio insuficiente.

### Acceso mediante SSH

**Ejecutar en:** el equipo desde el que se abrirá el navegador.

```bash
ssh -N -L 2283:127.0.0.1:2283 usuario@IP_DEL_SERVIDOR
```

Abrir:

```text
http://localhost:2283
```

Completar el asistente inicial y crear la cuenta administrativa.

---

## 6. Preparar la exportación de Google Photos

Antes de importar:

1. Reunir todas las partes de la exportación.
2. Conservar una copia original sin modificar.
3. Comprobar que los archivos comprimidos pueden leerse.
4. Identificar fotografías, vídeos y archivos auxiliares.
5. Seleccionar una muestra para validar el procedimiento.

Los archivos JSON de Google Takeout pueden contener información relevante para reconstruir metadatos. No eliminarlos durante una limpieza inicial.

### Comprobar un archivo ZIP

**Ejecutar en:** el equipo Linux donde se preparan las exportaciones.

Con `unzip` instalado:

```bash
unzip -t "/ruta/exportacion/takeout-001.zip"
```

La prueba revisa la integridad del archivo comprimido. No confirma que la exportación incluya toda la colección.

### Revisar el tamaño

```bash
du -sh "/ruta/exportacion"
```

Si se ha extraído una copia de trabajo:

```bash
find "/ruta/exportacion-extraida" -type f -print | head -n 30
```

Los nombres de archivo pueden revelar información personal. Revisar cualquier salida antes de incluirla en capturas.

---

## 7. Elegir el método de importación

### Subida desde la interfaz

Útil para una muestra pequeña de fotografías y vídeos.

Permite comprobar el funcionamiento básico antes de procesar una colección completa.

### Importación de Google Takeout

Una migración desde Takeout puede necesitar interpretar archivos auxiliares, fechas y álbumes.

La documentación de Immich remite a la herramienta comunitaria **immich-go** para este caso. Antes de utilizarla, revisar su compatibilidad con la versión del servidor y su documentación.

No se debe asumir que una subida genérica de imágenes reconstruirá toda la información de Google Photos.

### Procedimiento de validación

1. Importar una muestra representativa.
2. Revisar fechas y orientación.
3. Abrir fotografías y reproducir vídeos.
4. Comprobar álbumes y metadatos cuando el método los admita.
5. Revisar errores y elementos omitidos.
6. Continuar por lotes después de validar la muestra.

---

## 8. Comprobar la migración

| Elemento | Comprobación |
|---|---|
| Fotografías | Se abren y muestran correctamente |
| Vídeos | Se reproducen |
| Fechas | Coinciden con una muestra de los originales |
| Metadatos | Se conservan los campos esperados |
| Álbumes | Se reconstruyen cuando el método lo permite |
| Duplicados | Se revisa su tratamiento |
| Errores | Se registran archivos rechazados u omitidos |
| Procesamiento | Se revisan las tareas pendientes o fallidas |

El número de archivos de una exportación puede incluir JSON, duplicados y otros elementos. No debe compararse directamente con el total de fotografías de Immich sin clasificarlo.

Conservar el origen hasta completar las comprobaciones y disponer de una copia independiente.

---

## 9. Revisar recursos durante la importación

**Ejecutar en:** el sistema que aloja Docker.

```bash
sudo docker stats --no-stream
free -h
df -h
```

Desde la carpeta de Immich:

```bash
sudo docker compose ps
sudo docker compose logs --tail=100
```

La importación puede generar trabajo adicional después de terminar la transferencia.

### Diagnóstico inicial

| Síntoma | Qué revisar |
|---|---|
| Archivos rechazados | Formato, integridad y registros |
| Fechas incorrectas | Metadatos originales y método de importación |
| Miniaturas pendientes | Colas de procesamiento y recursos |
| Falta de espacio | Originales, temporales y contenido generado |
| Errores de escritura | Montajes y permisos |
| Aplicación sin respuesta | Estado de servicios, recursos y base de datos |

---

## 10. Backups y recuperación

Para una recuperación completa se deben proteger:

- Fotografías y vídeos.
- Base de datos.
- Configuración del despliegue.
- Archivos externos utilizados por la aplicación.

Un backup de la base de datos no contiene las fotografías y los vídeos.

Tampoco basta con conservar únicamente las imágenes para recuperar usuarios, relaciones y toda la organización de la aplicación.

### Relación con Proxmox

Si Immich está alojado en un LXC:

1. Revisar `rootfs` y los puntos de montaje.
2. Confirmar qué ubicaciones incluye el backup.
3. Preparar copias independientes para los datos excluidos.
4. Coordinar la consistencia entre archivos y base de datos.

No copiar directamente los archivos de PostgreSQL mientras está activo como si fueran una carpeta ordinaria de documentos. Utilizar un procedimiento de respaldo compatible.

**Pendiente de documentar:** una restauración completa en un entorno aislado, con comprobación de archivos y base de datos.

---

## 11. Evidencias para el proyecto

- Captura del servicio operativo con datos de prueba.
- Estado de los contenedores.
- Ejemplo de importación y comprobaciones.
- Incidencias encontradas y solución aplicada.
- Revisión de almacenamiento y cobertura del backup.

Ocultar rostros, ubicaciones, nombres de álbumes privados, credenciales y demás información personal.

---

## Conocimientos practicados

- Despliegue de aplicaciones con varios servicios.
- Configuración de variables y almacenamiento persistente.
- Preparación de una migración de datos.
- Validación de archivos y metadatos.
- Supervisión de recursos y registros.
- Revisión de la cobertura de las copias de seguridad.

---

## Referencias

- [Instalación de Immich con Docker Compose](https://docs.immich.app/install/docker-compose/)
- [Herramientas de importación de Immich](https://docs.immich.app/features/command-line-interface/)
- [immich-go](https://github.com/simulot/immich-go)
- [Backup y restauración de Immich](https://docs.immich.app/administration/backup-and-restore/)

---

[Volver al inicio](../README.md) · [Anterior: almacenamiento y backups](05-almacenamiento-backups.md)
