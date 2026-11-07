# 01 – Instalación y preparación de Proxmox VE

## Objetivo

Preparar una plataforma de virtualización y comprobar su funcionamiento antes de crear máquinas virtuales y contenedores LXC.

Los ejemplos utilizan valores genéricos y no muestran la configuración de la infraestructura personal.

---

## 1. Preparación del equipo

Antes de comenzar:

- Comprobar que el equipo admite virtualización por hardware.
- Habilitar la virtualización en BIOS/UEFI.
- Preparar el almacenamiento destinado a Proxmox.
- Conectar el equipo a la red mediante cable.
- Disponer de una memoria USB para el instalador.

> La instalación formatea el disco seleccionado. Es necesario respaldar los datos existentes y comprobar el destino antes de continuar.

---

## 2. Instalación de Proxmox VE

1. Descargar la ISO desde la [página oficial de Proxmox](https://www.proxmox.com/en/downloads).
2. Grabar la imagen en una memoria USB.
3. Arrancar el equipo desde el USB.
4. Seleccionar la instalación de Proxmox VE.
5. Elegir el disco de destino.
6. Configurar ubicación, zona horaria y teclado.
7. Definir la contraseña administrativa y el correo de contacto.
8. Configurar la interfaz de administración, nombre del host, IP, puerta de enlace y DNS.
9. Revisar el resumen e iniciar la instalación.
10. Reiniciar y retirar el USB.

La dirección de administración debe ser estable y no entrar en conflicto con otros dispositivos de la red.

---

## 3. Primer acceso

Desde un equipo con acceso a la red de administración:

```text
https://IP_DEL_HOST:8006
```

En una instalación estándar:

| Campo | Valor |
|---|---|
| Usuario | `root` |
| Autenticación | Linux PAM |
| Contraseña | La definida durante la instalación |

El certificado inicial es autofirmado. Antes de aceptar una excepción en el navegador, comprobar que la dirección corresponde al equipo instalado.

### Comprobaciones desde la interfaz

- El nodo aparece disponible.
- Se muestran CPU, memoria y almacenamiento.
- La fecha y la hora son correctas.
- No hay tareas fallidas pendientes de revisar.

---

## 4. Verificar la plataforma

**Ejecutar en:** host Proxmox, desde la opción **Shell** del nodo.

### Consultar la versión

```bash
pveversion
```

Para consultar también las versiones de los paquetes:

```bash
pveversion --verbose
```

Esta información permite identificar el entorno al investigar errores o revisar compatibilidades.

### Consultar los servicios principales

```bash
systemctl status pveproxy pvedaemon pvestatd --no-pager
```

| Servicio | Función |
|---|---|
| `pveproxy` | Acceso HTTPS a la interfaz y la API |
| `pvedaemon` | Operaciones de administración |
| `pvestatd` | Recogida del estado de los recursos |

Si algún servicio presenta errores:

```bash
journalctl -b -u pveproxy -u pvedaemon -u pvestatd \
  --no-pager -n 100
```

**Qué se aprende:** comprobar el estado de un servicio y consultar sus registros antes de modificar la configuración.

---

## 5. Comprobar la red

**Ejecutar en:** host Proxmox.

### Interfaces y direcciones

```bash
ip -br address
```

Permite identificar las interfaces físicas y los puentes de red.

En una instalación habitual, un puente como `vmbr0` conecta los entornos virtualizados con la red. La dirección de administración suele estar asignada al puente.

### Rutas

```bash
ip route
```

Comprobar la ruta hacia la red local y la puerta de enlace predeterminada, cuando corresponda.

### Resolución DNS

```bash
getent hosts download.proxmox.com
```

Si devuelve direcciones, el sistema ha podido resolver ese nombre. Esto no confirma por sí solo que la descarga de paquetes funcione.

### Puerto de la interfaz web

```bash
ss -ltnp 'sport = :8006'
```

Comprueba si existe un proceso escuchando en el puerto de administración.

**Qué se aprende:** diferenciar problemas de direccionamiento, rutas, DNS y disponibilidad del servicio.

---

## 6. Revisar recursos y almacenamiento

**Ejecutar en:** host Proxmox.

### Memoria disponible

```bash
free -h
```

Permite revisar la memoria antes de asignar recursos a máquinas virtuales o contenedores.

### Discos y particiones

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Muestra dispositivos, particiones, sistemas de archivos y puntos de montaje.

### Espacio de los sistemas de archivos

```bash
df -h
```

Muestra el espacio utilizado y disponible en los sistemas de archivos montados. No representa por sí solo toda la capacidad de almacenamientos como LVM-thin.

### Almacenamientos de Proxmox

```bash
pvesm status
```

Permite revisar el estado y la capacidad de los almacenamientos configurados.

Desde **Datacenter → Storage** se comprueba qué contenidos admite cada destino:

| Contenido | Uso |
|---|---|
| Imágenes ISO | Instalación de sistemas operativos |
| Plantillas LXC | Creación de contenedores |
| Discos de máquinas virtuales | Almacenamiento de las VMs |
| Sistemas de archivos de contenedores | Almacenamiento de los LXC |
| Backups | Copias de seguridad |

**Qué se aprende:** distinguir entre un disco físico, un sistema de archivos y un almacenamiento configurado en Proxmox.

---

## 7. Revisar repositorios y actualizaciones

Desde el nodo, acceder a **Updates → Repositories**.

Comprobar que los repositorios corresponden a la versión instalada y a la disponibilidad de una suscripción.

**Ejecutar en:** host Proxmox.

```bash
# Actualizar el índice de paquetes
apt update

# Consultar las actualizaciones disponibles
apt list --upgradable
```

Si aparecen errores de repositorios, resolverlos antes de continuar. No mezclar repositorios de distintas versiones de Debian o Proxmox.

Una vez revisados los repositorios y el impacto de la actualización:

```bash
apt full-upgrade
```

Leer los cambios propuestos antes de confirmar. Si es necesario reiniciar, planificarlo considerando las máquinas virtuales y los contenedores alojados.

---

## 8. Validación final

| Comprobación | Resultado esperado |
|---|---|
| Interfaz web | Acceso e inicio de sesión disponibles |
| Servicios principales | Activos |
| Red de administración | Dirección estable y sin conflictos |
| DNS | Resolución de nombres operativa |
| Almacenamiento | Destinos disponibles y con capacidad suficiente |
| Repositorios | Compatibles con la versión instalada |
| Actualización del índice | Sin errores pendientes de resolver |

Estas comprobaciones establecen la base para crear y administrar los primeros entornos virtualizados.

---

## Conocimientos practicados

- Instalación y preparación de Proxmox VE.
- Configuración inicial de acceso y red.
- Comprobación de servicios con `systemctl`.
- Consulta de registros con `journalctl`.
- Diagnóstico básico de red y DNS.
- Identificación de discos y almacenamientos.
- Revisión de repositorios y actualizaciones.

---

[Volver al inicio](../README.md) · [Resumen del proyecto](00-overview.md)
