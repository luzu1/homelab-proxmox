# 05 – Almacenamiento y copias de seguridad

## Objetivo

Revisar el almacenamiento de los entornos virtualizados, configurar copias de seguridad y comprobar qué datos quedan protegidos.

En el proyecto se han configurado tareas de backup y revisado copias de contenedores. La prueba completa de restauración queda pendiente de documentar.

Los identificadores, nombres y rutas de este capítulo son ejemplos genéricos.

---

## 1. Diferenciar almacenamiento, snapshots y backups

| Concepto | Función |
|---|---|
| Almacenamiento del entorno | Aloja los discos y datos utilizados por una VM o un LXC |
| Snapshot | Conserva un estado del entorno y depende del almacenamiento que lo contiene |
| Backup | Genera una copia recuperable de los elementos incluidos |
| Restauración | Recupera un entorno o sus datos desde una copia |

Una instantánea no sustituye una copia de seguridad independiente.

Asimismo, guardar backups en otro disco del mismo equipo no protege frente a todos los fallos del host, robos o daños físicos.

---

## 2. Revisar discos y sistemas de archivos

**Ejecutar en:** host Proxmox.

### Identificar dispositivos

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Permite distinguir discos, particiones, sistemas de archivos y puntos de montaje.

### Revisar el espacio disponible

```bash
df -h
```

Muestra el uso de los sistemas de archivos montados. No representa por sí solo toda la capacidad de almacenamientos como LVM-thin.

### Revisar los montajes

```bash
findmnt
```

Antes de utilizar un directorio como destino de backups, comprobar que pertenece al dispositivo previsto.

Para consultar qué sistema de archivos contiene una ruta:

```bash
findmnt --target /mnt/backup-lab
```

La ruta es ilustrativa y debe existir previamente.

Si se esperaba un disco independiente y aparece el sistema de archivos raíz, revisar el montaje antes de generar copias.

---

## 3. Revisar los almacenamientos de Proxmox

**Ejecutar en:** host Proxmox.

```bash
pvesm status
```

Comprobar:

- Identificador del almacenamiento.
- Tipo.
- Estado.
- Capacidad y espacio disponible.

Desde **Datacenter → Storage** se revisan los tipos de contenido admitidos.

| Tipo de contenido | Utilidad |
|---|---|
| ISO | Instalación de sistemas operativos |
| Plantillas de contenedores | Creación de LXC |
| Discos de VM | Almacenamiento de máquinas virtuales |
| Sistemas de archivos de LXC | Almacenamiento de contenedores |
| Backups | Copias recuperables de los entornos |

**Aprendizaje:** distinguir la ubicación física de los datos del identificador utilizado por Proxmox.

---

## 4. Registrar un destino de backups

Este ejemplo supone que el disco ya está preparado y montado de forma persistente.

Desde **Datacenter → Storage → Add → Directory**:

1. Introducir un identificador, por ejemplo `backup-lab`.
2. Indicar el directorio montado.
3. Seleccionar el contenido de copias de seguridad.
4. Limitar los nodos que pueden utilizarlo cuando corresponda.
5. Guardar y comprobar su disponibilidad.

Crear una carpeta no equivale a montar un disco. El destino debe revisarse también después de reiniciar el host.

### Comprobación desde consola

```bash
pvesm status
```

El destino debe aparecer disponible antes de programar tareas.

---

## 5. Comprobar qué datos se incluyen

Antes de crear una copia, revisar los discos y montajes del entorno.

**Ejecutar en:** host Proxmox.

Los identificadores `200` y `201` representan una VM y un LXC de laboratorio.

### Máquina virtual

```bash
qm config 200
```

Revisar sus discos y si alguno está excluido de las copias.

### Contenedor LXC

```bash
pct config 201
```

Revisar:

- Disco raíz: `rootfs`.
- Puntos de montaje adicionales: `mp0`, `mp1`, etc.
- Opciones de inclusión en el backup.
- Directorios o dispositivos externos al almacenamiento administrado por Proxmox.

### Consideraciones para LXC

Los puntos de montaje adicionales requieren revisar su inclusión.

Los montajes bind y de dispositivos no se respaldan mediante el backup del contenedor. Sus datos necesitan una copia independiente.

### Aplicaciones Docker dentro del entorno

Un volumen Docker no queda protegido automáticamente por el hecho de llamarse «volumen».

Su cobertura depende de dónde estén almacenados físicamente sus datos y de si esa ubicación está incluida en la copia del entorno.

**Aprendizaje:** comprobar los datos reales protegidos, no solo que el backup de la VM o del LXC termine correctamente.

---

## 6. Elegir el modo de backup

| Modo | Consideraciones |
|---|---|
| Snapshot | Permite copiar con una interrupción reducida; su funcionamiento y requisitos dependen de si se trata de una VM o un LXC |
| Suspend | Implica una suspensión temporal y tiene un comportamiento diferente según el entorno |
| Stop | Utiliza la parada del entorno para realizar la copia, con impacto en la disponibilidad |

En LXC, el modo snapshot requiere soporte adecuado del almacenamiento para los volúmenes incluidos.

El modo se elige según los requisitos de disponibilidad y consistencia de la aplicación.

Las bases de datos pueden necesitar procedimientos propios de respaldo o coordinación para garantizar una recuperación coherente.

---

## 7. Crear una copia manual

**Ejecutar en:** host Proxmox.

El almacenamiento `backup-lab` debe existir y admitir backups.

### Ejemplo de copia de un LXC

```bash
vzdump 201 \
  --storage backup-lab \
  --mode snapshot \
  --compress zstd
```

### Ejemplo de copia de una VM

```bash
vzdump 200 \
  --storage backup-lab \
  --mode snapshot \
  --compress zstd
```

Estos ejemplos están orientados a un almacenamiento de archivos de backup.

### Qué revisar durante la tarea

- Entorno seleccionado.
- Destino utilizado.
- Volúmenes incluidos y excluidos.
- Errores o avisos.
- Resultado final.

Si snapshot no es compatible con la configuración del LXC, revisar el motivo y planificar otro modo considerando la interrupción necesaria.

---

## 8. Programar las copias

Desde **Datacenter → Backup**:

1. Crear una tarea.
2. Seleccionar el nodo y el almacenamiento.
3. Elegir las VMs y los contenedores incluidos.
4. Definir la frecuencia.
5. Seleccionar el modo de copia.
6. Configurar la retención.
7. Revisar las notificaciones disponibles.
8. Guardar y comprobar la primera ejecución.

### Criterios de planificación

- Evitar que las tareas compitan innecesariamente por disco o red.
- Considerar cuánto cambian los datos.
- Definir cuánta información se podría perder entre copias.
- Reservar capacidad para nuevas copias.
- Revisar el resultado de las ejecuciones.

Una tarea programada no demuestra que las copias estén terminando correctamente.

---

## 9. Definir la retención

La retención establece qué versiones se conservan.

Como ejemplo de planificación, se podría valorar conservar:

- Copias recientes para recuperar cambios de los últimos días.
- Copias semanales para disponer de un historial más amplio.
- Copias mensuales cuando exista capacidad y una necesidad concreta.

La cantidad se ajusta al tamaño de los datos, la frecuencia de cambio y el espacio disponible.

Antes de aplicar una política de eliminación, revisar qué copias dejarían de conservarse.

---

## 10. Revisar el resultado

### Desde la interfaz

Abrir la tarea terminada y consultar su registro.

Comprobar que el resultado es correcto y revisar los avisos aunque no hayan provocado un fallo.

### Desde consola

**Ejecutar en:** host Proxmox.

```bash
pvesm list backup-lab --content backup
```

Permite localizar las copias registradas en el destino.

También se revisa la capacidad restante:

```bash
pvesm status
```

### Evidencias útiles

| Evidencia | Qué demuestra |
|---|---|
| Configuración de la tarea | Qué se pretende copiar y con qué frecuencia |
| Registro de ejecución | Qué ocurrió durante la copia |
| Archivo o entrada de backup | Que existe una copia registrada |
| Prueba de restauración | Que se pudo recuperar y validar el entorno |

La existencia de un archivo no garantiza que la aplicación pueda recuperarse correctamente.

---

## 11. Preparar una prueba de restauración

**Estado: pendiente de documentar con resultados reales.**

La prueba debe utilizar un identificador nuevo y almacenamiento con capacidad suficiente.

### Procedimiento previsto

1. Seleccionar una copia concreta.
2. Registrar su fecha y el entorno de origen.
3. Elegir un identificador libre.
4. Restaurar sin sobrescribir el entorno original.
5. Mantener el entorno restaurado apagado mientras se revisa su configuración.
6. Aislar su red antes del primer arranque.
7. Revisar montajes y referencias a recursos compartidos.
8. Iniciar y comprobar el sistema.
9. Validar los datos y la aplicación.
10. Registrar el tiempo y las incidencias.

El aislamiento evita duplicar direcciones o ejecutar tareas, integraciones y automatizaciones contra servicios reales.

### Comprobaciones de recuperación

- El sistema arranca.
- Los discos y datos esperados están presentes.
- La aplicación inicia.
- Se puede acceder con las credenciales previstas.
- Las funciones principales responden.
- Se identifican los datos que requieren recuperación independiente.

---

## 12. Ampliaciones previstas

- Registrar una restauración completa.
- Documentar el respaldo de datos excluidos.
- Mantener una copia adicional fuera del host.
- Revisar periódicamente la retención y la capacidad.
- Documentar la recuperación de la configuración del host Proxmox.

Los backups de las VMs y los contenedores no equivalen a una copia completa de la configuración del host.

---

## Conocimientos practicados

- Identificación de discos y montajes.
- Revisión de almacenamientos de Proxmox.
- Selección de destinos de backup.
- Revisión de los datos incluidos y excluidos.
- Ejecución de copias con `vzdump`.
- Programación de tareas.
- Comprobación de registros y capacidad.
- Planificación de pruebas de recuperación.

---

## Referencias

- [Documentación de Proxmox VE](https://pve.proxmox.com/pve-docs/)
- [Backup y restauración: documentación oficial](https://github.com/proxmox/pve-docs/blob/master/vzdump.adoc)

---

[Volver al inicio](../README.md) · [Anterior: monitorización con Uptime Kuma](04-monitorizacion-uptime-kuma.md)
