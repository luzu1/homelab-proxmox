# 08 – Diagnóstico y mantenimiento

## Objetivo

Documentar un método de diagnóstico para identificar problemas en entornos virtualizados y servicios autohospedados.

El procedimiento combina comprobaciones de recursos, red, servicios y registros. Los comandos son ejemplos genéricos; no representan un informe de incidencias ya verificadas.

---

## 1. Método de trabajo

Antes de modificar una configuración:

1. Describir el síntoma.
2. Identificar qué servicios están afectados.
3. Anotar cuándo comenzó el problema.
4. Revisar cambios recientes.
5. Recoger evidencias.
6. Formular una posible causa.
7. Aplicar un cambio concreto.
8. Comprobar el resultado.

Cambiar varios elementos simultáneamente dificulta saber qué causó o resolvió el fallo.

### Información inicial

| Pregunta | Utilidad |
|---|---|
| ¿Qué falla exactamente? | Delimitar el problema |
| ¿Desde dónde falla? | Diferenciar aplicación, acceso y conectividad |
| ¿Afecta a todos los usuarios? | Determinar el alcance |
| ¿Cuándo funcionó por última vez? | Acotar la revisión de registros |
| ¿Qué cambió recientemente? | Identificar posibles causas |
| ¿Se puede reproducir? | Preparar una comprobación fiable |

---

## 2. Identificar la capa afectada

| Capa | Elementos que revisar |
|---|---|
| Host Proxmox | Recursos físicos, almacenamiento y servicios de virtualización |
| VM o LXC | Arranque, recursos asignados y sistema operativo |
| Docker | Estado de contenedores, redes y montajes |
| Aplicación | Configuración, dependencias y registros |
| Acceso | DNS, rutas, puertos, proxy y certificados |

Los comandos deben ejecutarse en la capa correspondiente.

Un contenedor Docker detenido y una VM sin conectividad requieren investigaciones distintas.

---

## 3. Revisar el host Proxmox

**Ejecutar en:** host Proxmox.

### Recursos generales

```bash
uptime
free -h
df -h
pvesm status
```

Estas consultas muestran carga, memoria, espacio de los sistemas de archivos y estado de los almacenamientos.

La carga del sistema debe interpretarse considerando el número de CPU y la posible espera de operaciones de disco.

### Estado de los entornos

```bash
qm list
pct list
```

### Configuración de un entorno concreto

Los identificadores siguientes son ejemplos:

```bash
qm config 200
pct config 201
```

### Servicios de administración

```bash
systemctl status pveproxy pvedaemon pvestatd --no-pager
```

Antes de reiniciar un servicio, consultar los registros y determinar qué función está afectada.

---

## 4. Revisar una VM o un LXC Linux

**Ejecutar en:** el sistema invitado afectado.

### Identificar el sistema y sus recursos

```bash
cat /etc/os-release
uptime
free -h
df -h
df -i
```

`df -i` permite comprobar los inodos. Un sistema de archivos puede quedarse sin capacidad para crear nuevos archivos aunque todavía tenga espacio en bytes.

### Identificar procesos con mayor consumo

```bash
ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head -n 15
```

La salida es una observación puntual. Conviene contrastarla con el comportamiento durante el fallo.

### Servicios fallidos

En sistemas con systemd:

```bash
systemctl --failed
```

Para un servicio concreto:

```bash
systemctl status NOMBRE_DEL_SERVICIO --no-pager
```

Sustituir el nombre por el servicio que se está investigando.

---

## 5. Consultar registros

**Ejecutar en:** el sistema Linux afectado.

### Avisos y errores del arranque actual

```bash
journalctl -b -p warning --no-pager -n 100
```

### Registros de un servicio

```bash
journalctl -u NOMBRE_DEL_SERVICIO \
  --since "30 minutes ago" \
  --no-pager
```

### Mensajes recientes del kernel

```bash
sudo journalctl -k -b --no-pager -n 100
```

Los mensajes del kernel pueden aportar información sobre dispositivos, errores de entrada/salida o procesos terminados por falta de memoria.

### Qué registrar

- Hora del evento.
- Servicio o proceso.
- Mensaje de error.
- Frecuencia.
- Relación con el síntoma observado.

Un aviso aislado no demuestra que sea la causa del problema.

---

## 6. Diagnosticar conectividad

**Ejecutar en:** el equipo desde el que falla el acceso.

### Interfaces y rutas

```bash
ip -br address
ip route
```

### Resolución de nombres

```bash
getent hosts servicio.example.com
```

El dominio es ilustrativo.

### Comprobación HTTP

```bash
curl --output /dev/null \
  --silent \
  --show-error \
  --max-time 10 \
  --write-out 'HTTP: %{http_code}\nTiempo: %{time_total}s\n' \
  https://servicio.example.com
```

Esta prueba refleja únicamente el acceso desde el equipo donde se ejecuta.

### Puertos en escucha

**Ejecutar en:** el sistema que debe prestar el servicio.

```bash
sudo ss -tulnp
```

Revisar si el servicio escucha en:

- La interfaz local.
- Una dirección concreta.
- Todas las interfaces.
- El puerto esperado.

### Interpretación inicial

| Síntoma | Posibles comprobaciones |
|---|---|
| El nombre no resuelve | DNS y configuración del cliente |
| Conexión rechazada | Servicio, puerto y rechazo activo |
| Tiempo de espera agotado | Ruta, filtrado o falta de respuesta |
| Error TLS | Nombre, vigencia y confianza del certificado |
| HTTP 401/403 | Autenticación y permisos |
| HTTP 500 | Registros de la aplicación |
| HTTP 502/504 | Proxy y comunicación con el servicio de destino |

---

## 7. Diagnosticar aplicaciones Docker

**Ejecutar en:** el sistema que aloja Docker.

### Estado general

```bash
sudo docker ps -a
sudo docker stats --no-stream
sudo docker system df
```

### Estado de una aplicación Compose

Desde la carpeta del proyecto:

```bash
sudo docker compose ps -a
sudo docker compose logs --since=30m --tail=200
```

### Revisar un contenedor concreto

Sustituir `NOMBRE_CONTENEDOR` por el nombre obtenido en el listado:

```bash
sudo docker inspect NOMBRE_CONTENEDOR \
  --format '{{json .State}}'
```

Revisar su número de reinicios:

```bash
sudo docker inspect NOMBRE_CONTENEDOR \
  --format '{{.RestartCount}}'
```

Un reinicio repetido es un síntoma. La causa puede estar en la configuración, los permisos, las dependencias o los recursos.

### Validar Compose

```bash
sudo docker compose config --quiet
```

Esta comprobación detecta problemas de configuración, pero no garantiza que la aplicación funcione.

---

## 8. Revisar almacenamiento y permisos

**Ejecutar en:** el sistema que accede a los datos.

```bash
df -h
df -i
findmnt
```

Para una ruta concreta:

```bash
ls -ld /ruta/datos
stat /ruta/datos
findmnt --target /ruta/datos
```

Comprobar:

- Que la ruta existe.
- Que corresponde al almacenamiento previsto.
- Que el usuario del servicio tiene el acceso necesario.
- Que el montaje no está en modo de solo lectura.
- Que hay espacio e inodos disponibles.

No utilizar permisos globales como `chmod -R 777` para ocultar un problema de acceso. Identificar el usuario y los permisos requeridos.

No eliminar datos, volúmenes o registros sin revisar su función y su respaldo.

---

## 9. Mantenimiento planificado

### Antes del cambio

- Registrar la versión actual.
- Revisar los cambios de la nueva versión.
- Comprobar compatibilidad.
- Confirmar que existe una copia recuperable.
- Definir cómo volver al estado anterior.
- Prever la interrupción necesaria.

### Durante el cambio

- Modificar un componente cada vez cuando sea posible.
- Observar los resultados de la tarea.
- Registrar errores.
- Evitar encadenar actualizaciones sin comprobaciones.

### Después del cambio

- Revisar el estado de los servicios.
- Comprobar acceso y funciones principales.
- Consultar los registros.
- Confirmar que los monitores se recuperan.
- Documentar el resultado.

Un reinicio correcto no sustituye una prueba funcional.

---

## 10. Ejemplo de investigación

### Síntoma

Un monitor informa de que una aplicación web no responde.

### Secuencia de comprobación

1. Revisar la hora y el error del monitor.
2. Confirmar que la VM o el LXC está iniciado.
3. Comprobar espacio, memoria y conectividad.
4. Revisar el estado de la aplicación.
5. Consultar sus registros.
6. Comprobar el puerto y la respuesta HTTP.
7. Revisar el proxy si el acceso directo funciona.
8. Aplicar la corrección correspondiente a las evidencias.
9. Confirmar recuperación desde el punto donde se detectó el fallo.

### Criterio de cierre

La incidencia se considera resuelta cuando el síntoma deja de reproducirse, la función afectada responde y se han comprobado los posibles efectos del cambio.

Este ejemplo describe una metodología; no atribuye una causa sin evidencias.

---

## Conocimientos practicados

- Diagnóstico por capas.
- Revisión de recursos y almacenamiento.
- Consulta de registros de Linux y Docker.
- Comprobación de conectividad y servicios.
- Investigación de permisos y montajes.
- Planificación del mantenimiento.
- Documentación de incidencias y validación de soluciones.

---

[Volver al inicio](../README.md) · [Anterior: Home Assistant y Zigbee/MQTT](07-home-assistant-zigbee-mqtt.md)
