# 02 – Máquinas virtuales y contenedores LXC

## Objetivo

Crear y administrar entornos virtualizados en Proxmox VE, asignar recursos y comprobar su funcionamiento desde la interfaz web y la consola.

Los identificadores utilizados en los ejemplos son ficticios:

- `200`: máquina virtual de laboratorio.
- `201`: contenedor LXC de laboratorio.

Los comandos de administración deben ejecutarse sobre entornos existentes y con sus identificadores correspondientes.

---

## 1. Diferencias entre una VM y un contenedor LXC

| Característica | Máquina virtual | Contenedor LXC |
|---|---|---|
| Sistema | Ejecuta su propio kernel | Comparte el kernel del host |
| Sistemas compatibles | Linux, Windows y otros compatibles | Linux |
| Recursos | Requiere recursos para un sistema completo | Generalmente consume menos recursos |
| Aislamiento | Mediante virtualización de hardware | Mediante mecanismos del kernel |
| Usos habituales | Sistemas completos y aplicaciones que requieren una VM | Servicios y entornos Linux ligeros |

La elección depende de los requisitos de la aplicación, el aislamiento necesario y los recursos disponibles.

En el proyecto se han utilizado ambos tipos de entorno, incluyendo una VM para Home Assistant OS y contenedores LXC para servicios Linux.

---

## 2. Preparar los archivos de instalación

### Para máquinas virtuales

Desde un almacenamiento que admita imágenes ISO:

1. Abrir **ISO Images**.
2. Subir la imagen del sistema operativo.
3. Comprobar que aparece disponible.

### Para contenedores LXC

Desde un almacenamiento que admita plantillas:

1. Abrir **CT Templates**.
2. Seleccionar **Templates**.
3. Descargar una plantilla Linux compatible con el servicio previsto.

**Qué se aprende:** diferenciar una ISO de instalación de una plantilla de contenedor.

---

## 3. Crear una máquina virtual

Desde la interfaz de Proxmox, seleccionar **Create VM**.

### Configuración general

1. Elegir el nodo.
2. Asignar un identificador libre.
3. Introducir un nombre descriptivo.
4. Seleccionar la ISO.
5. Configurar las opciones de sistema según los requisitos del invitado.
6. Seleccionar el almacenamiento y el tamaño del disco.
7. Asignar CPU y memoria.
8. Seleccionar el puente de red apropiado.
9. Revisar el resumen.

Los valores predeterminados no son adecuados para todos los sistemas. Algunas aplicaciones requieren ajustes específicos de firmware, controladores o discos.

### Instalación del sistema operativo

1. Iniciar la VM.
2. Abrir **Console**.
3. Completar el instalador del sistema.
4. Retirar la ISO del lector virtual al finalizar.
5. Comprobar el arranque desde el disco instalado.

> Algunas aplicaciones, como Home Assistant OS, se despliegan mediante una imagen de disco preparada. Ese procedimiento es diferente al de una instalación desde ISO.

---

## 4. Administrar una VM desde consola

**Ejecutar en:** host Proxmox.

### Listar las máquinas virtuales

```bash
qm list
```

Permite identificar las VMs y consultar su estado.

### Revisar la configuración

```bash
qm config 200
```

Muestra los recursos y dispositivos configurados, como memoria, CPU, discos y conexiones de red.

### Consultar el estado

```bash
qm status 200
```

### Iniciar la VM

```bash
qm start 200
```

### Solicitar un apagado ordenado

```bash
qm shutdown 200
```

El apagado depende de que el sistema invitado pueda atender la solicitud. Si no termina, revisar su consola antes de forzar una parada.

**Qué se aprende:** administrar el ciclo de vida de una VM sin depender exclusivamente de la interfaz gráfica.

---

## 5. Crear un contenedor LXC

Desde la interfaz de Proxmox, seleccionar **Create CT**.

### Configuración inicial

1. Elegir el nodo y un identificador libre.
2. Asignar un nombre al contenedor.
3. Configurar las credenciales o la clave pública SSH.
4. Mantener el contenedor sin privilegios cuando la aplicación lo permita.
5. Seleccionar la plantilla descargada.
6. Asignar almacenamiento, CPU y memoria.
7. Configurar el puente de red y el direccionamiento.
8. Revisar la configuración DNS.
9. Confirmar la creación e iniciar el contenedor.

Un contenedor sin privilegios reduce los permisos que sus usuarios pueden ejercer sobre el host.

Las funciones adicionales y el acceso a dispositivos se habilitan solo cuando el servicio lo requiere.

---

## 6. Administrar un LXC desde consola

**Ejecutar en:** host Proxmox.

### Listar los contenedores

```bash
pct list
```

### Revisar la configuración

```bash
pct config 201
```

Permite consultar recursos, red y puntos de montaje.

### Consultar el estado

```bash
pct status 201
```

### Iniciar el contenedor

```bash
pct start 201
```

### Abrir una sesión dentro del contenedor

```bash
pct enter 201
```

A partir de ese momento, los comandos se ejecutan dentro del contenedor.

Para regresar al host:

```bash
exit
```

### Ejecutar una comprobación sin abrir una sesión

```bash
pct exec 201 -- cat /etc/os-release
```

### Solicitar un apagado ordenado

```bash
pct shutdown 201
```

**Qué se aprende:** distinguir entre administrar el contenedor desde Proxmox y trabajar dentro de su sistema operativo.

---

## 7. Comprobar el sistema invitado

**Ejecutar en:** consola de una VM Linux o dentro de un contenedor LXC.

### Identificar el sistema

```bash
cat /etc/os-release
```

### Revisar memoria y almacenamiento

```bash
free -h
df -h
```

### Comprobar interfaces y rutas

```bash
ip -br address
ip route
```

### Verificar resolución de nombres

```bash
getent hosts debian.org
```

### Detectar servicios fallidos

En sistemas que utilizan systemd:

```bash
systemctl --failed
```

Estas comprobaciones permiten detectar problemas del entorno antes de instalar una aplicación.

---

## 8. Asignación y ajuste de recursos

Antes de aumentar recursos, revisar el consumo y determinar qué limita al servicio.

| Recurso | Qué revisar |
|---|---|
| CPU | Uso sostenido y carga del sistema |
| Memoria | Memoria disponible, uso de swap y posibles errores por falta de memoria |
| Disco | Espacio disponible y crecimiento de los datos |
| Red | Conectividad, rutas y respuesta del servicio |

### Máquinas virtuales

Los recursos se revisan desde **Hardware**.

### Contenedores LXC

Los recursos se revisan desde **Resources**.

Algunos cambios requieren apagar o reiniciar el entorno. La interfaz puede mostrar modificaciones pendientes de aplicar.

> Ampliar un disco virtual no siempre amplía automáticamente la partición y el sistema de archivos del invitado. Es necesario comprobar ambas capas.

---

## 9. Arranque automático

Proxmox permite configurar si un entorno debe iniciarse al arrancar el host.

Desde las opciones de la VM o del contenedor:

1. Revisar **Start at boot**.
2. Configurar el orden y el retraso de inicio cuando existan dependencias.
3. Comprobar el comportamiento en una prueba de mantenimiento planificada.

El orden de inicio ayuda a organizar dependencias, pero no garantiza por sí solo que una aplicación esté preparada para recibir conexiones.

---

## 10. Diagnóstico de problemas habituales

| Problema | Comprobaciones iniciales |
|---|---|
| La VM o el LXC no inicia | Registro de la tarea, almacenamiento disponible y configuración |
| El invitado no tiene conectividad | Interfaz, puente asignado, dirección IP y rutas |
| No resuelve nombres | Configuración DNS y acceso al servidor DNS |
| La aplicación no responde | Estado del servicio, puertos y registros |
| El entorno funciona lentamente | CPU, memoria y almacenamiento |
| Un cambio no aparece aplicado | Opciones pendientes y necesidad de reinicio |

### Registros del sistema invitado

**Ejecutar en:** VM o LXC Linux con systemd.

```bash
journalctl -b -p warning --no-pager -n 100
```

### Puertos en escucha

```bash
ss -tuln
```

Un entorno encendido no garantiza que la aplicación funcione. La comprobación debe incluir el servicio y su acceso desde el punto previsto.

---

## 11. Validación final

- El entorno aparece con el nombre y el identificador previstos.
- El sistema inicia correctamente.
- CPU, memoria y almacenamiento corresponden a lo configurado.
- La conectividad funciona según el diseño del laboratorio.
- La resolución DNS está operativa cuando se necesita.
- El apagado ordenado funciona.
- La política de arranque automático es adecuada para el servicio.

---

## Conocimientos practicados

- Diferencias entre virtualización y contenedores de sistema.
- Creación de VMs y contenedores LXC.
- Asignación y revisión de recursos.
- Administración con `qm` y `pct`.
- Acceso a la consola de los sistemas invitados.
- Comprobaciones básicas de Linux y red.
- Diagnóstico inicial de fallos.
- Gestión del arranque y apagado.

---

[Volver al inicio](../README.md) · [Anterior: instalación y preparación](01-instalacion-proxmox.md)
