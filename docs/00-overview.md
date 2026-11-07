# 00 – Resumen del proyecto

## Objetivo

Practicar virtualización y administración de sistemas mediante el despliegue y mantenimiento de servicios autohospedados con Proxmox VE, Linux y Docker.

El proyecto documenta los procedimientos utilizados, los conocimientos adquiridos y las comprobaciones necesarias para administrar estos servicios.

---

## Tecnologías utilizadas

| Tecnología | Aplicación en el proyecto |
|---|---|
| Proxmox VE | Administración de entornos virtualizados |
| Máquinas virtuales | Ejecución de sistemas operativos invitados |
| LXC | Creación y administración de contenedores Linux |
| Linux | Administración de servicios, recursos, permisos y almacenamiento |
| Docker Compose | Definición y despliegue de aplicaciones en contenedores |
| Portainer | Administración visual de entornos Docker |
| Uptime Kuma | Monitorización de disponibilidad |
| Immich | Gestión de fotografías y vídeos |
| Home Assistant OS | Integración y administración de dispositivos domóticos |
| Zigbee y MQTT | Conectividad de dispositivos y mensajería entre servicios |

---

## Trabajo realizado

### Virtualización y administración

- Preparación de una plataforma Proxmox VE.
- Creación y administración de máquinas virtuales y contenedores LXC.
- Asignación de CPU, memoria y almacenamiento.
- Gestión del arranque, apagado y acceso a los entornos.
- Comprobación del estado de los sistemas desde la interfaz web y la consola.

### Despliegue de aplicaciones

- Administración de aplicaciones autohospedadas.
- Trabajo con contenedores Docker y archivos Docker Compose.
- Gestión de la persistencia de datos.
- Revisión del estado y los registros de los servicios.
- Uso de Portainer como herramienta de administración.

### Monitorización

- Uso de Uptime Kuma para comprobar la disponibilidad de servicios.
- Revisión del estado de los monitores.
- Comprobaciones de conectividad y respuesta de las aplicaciones.
- Consulta de registros para investigar fallos.

### Almacenamiento y backups

- Administración del almacenamiento utilizado por los entornos virtualizados.
- Revisión del espacio disponible y de los puntos de montaje.
- Configuración de copias de seguridad programadas en Proxmox.
- Ejecución y revisión de copias de contenedores.
- Identificación de aspectos pendientes de validar mediante pruebas de restauración.

### Servicios y migración de datos

- Despliegue y administración de Immich.
- Despliegue de Home Assistant OS.
- Integración de dispositivos y servicios Zigbee/MQTT.

---

## Conocimientos adquiridos

### Diferenciar las capas de administración

El trabajo con esta plataforma requiere distinguir dónde se ejecuta cada acción:

| Capa | Responsabilidad |
|---|---|
| Host Proxmox | Administrar la virtualización y los recursos del host |
| VM o contenedor LXC | Administrar el sistema y los servicios del entorno invitado |
| Docker | Administrar los contenedores de aplicaciones |
| Aplicación | Configurar sus funciones, usuarios y datos |

Esta separación permite seleccionar las herramientas adecuadas y evitar ejecutar comandos en el entorno equivocado.

### Comprobar el funcionamiento de un servicio

La validación combina varias comprobaciones:

- El entorno virtualizado está iniciado.
- Existen recursos suficientes.
- El proceso o contenedor de la aplicación está activo.
- El servicio responde desde el punto de acceso esperado.
- Los registros no muestran errores que impidan su funcionamiento.

### Mantener los datos de las aplicaciones

Se ha trabajado la diferencia entre el ciclo de vida de un contenedor y la persistencia de sus datos.

También se ha identificado la importancia de revisar qué información incluye cada copia de seguridad y qué datos necesitan un respaldo independiente.

### Documentar para poder reproducir

Los procedimientos deben indicar:

- El objetivo de la tarea.
- Los requisitos previos.
- Dónde se ejecutan los comandos.
- Qué función cumple cada paso.
- Cómo se comprueba el resultado.

---

## Organización de la documentación

| Capítulo | Contenido |
|---|---|
| 01 – Instalación y preparación | Preparación de Proxmox VE y comprobaciones iniciales |
| 02 – Máquinas virtuales y LXC | Creación, configuración y administración de entornos |
| 03 – Docker y Portainer | Despliegue y gestión de aplicaciones |
| 04 – Uptime Kuma | Monitorización de disponibilidad |
| 05 – Almacenamiento y backups | Gestión de almacenamiento y copias de seguridad |
| 06 – Immich | Despliegue y migración de fotografías |
| 07 – Home Assistant y Zigbee/MQTT | Domótica e integración de servicios |
| 08 – Diagnóstico y mantenimiento | Investigación de errores y comprobaciones |

Los ejemplos utilizan nombres, identificadores y rutas genéricas. Las capturas y configuraciones publicadas deben omitir credenciales y datos personales.

---


[Volver al inicio](../README.md)
