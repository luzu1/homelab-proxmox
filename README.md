# Homelab Proxmox VE (Linux + LXC + Docker)

**Objetivo:**

Desarrollar conocimientos prácticos en virtualización y administración de sistemas mediante Proxmox VE, máquinas virtuales, contenedores LXC y aplicaciones Docker.

El proyecto recoge el trabajo realizado en despliegue de servicios, gestión de almacenamiento, monitorización y copias de seguridad. La documentación utiliza ejemplos genéricos para explicar los procedimientos sin exponer la configuración de la infraestructura personal.

![Proxmox](https://img.shields.io/badge/Proxmox-E57000?style=for-the-badge&logo=proxmox&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Portainer](https://img.shields.io/badge/Portainer-13BEF9?style=for-the-badge&logo=portainer&logoColor=white)

---

## Documentación

- **00 - Resumen del proyecto**: Objetivos, tecnologías utilizadas, conocimientos adquiridos y alcance de las prácticas.

- **01 - Instalación y preparación de Proxmox VE**: Instalación de la plataforma, configuración inicial y comprobaciones del sistema, la red y el almacenamiento.

- **02 - Máquinas virtuales y contenedores LXC**: Creación de entornos, asignación de recursos, administración desde la interfaz web y comandos de gestión con `qm` y `pct`.

- **03 - Docker y administración con Portainer**: Despliegue de aplicaciones, archivos Docker Compose, persistencia de datos, revisión de registros y administración de contenedores.

- **04 - Monitorización con Uptime Kuma**: Configuración de monitores, comprobación de disponibilidad y diagnóstico de servicios que dejan de responder.

- **05 - Almacenamiento y copias de seguridad**: Revisión de discos y puntos de montaje, configuración de tareas de backup, comprobación de resultados y consideraciones para la recuperación.

- **06 - Immich y migración de fotografías**: Despliegue del servicio, organización del almacenamiento, migración desde Google Photos y comprobaciones del contenido importado.

- **07 - Home Assistant y servicios Zigbee/MQTT**: Despliegue de Home Assistant OS, integración de dispositivos y administración de los servicios de comunicación.

- **08 - Diagnóstico y mantenimiento**: Comprobación de recursos, consulta de logs y documentación de incidencias, soluciones aplicadas y verificaciones finales.

---

## Proyecto relacionado

[Nextcloud en Raspberry Pi 5](https://github.com/luzu1/rpi5-nextcloud): despliegue y administración de una nube privada con Docker, acceso remoto y procedimientos de mantenimiento.

