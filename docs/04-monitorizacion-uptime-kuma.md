# 04 – Monitorización con Uptime Kuma

## Objetivo

Monitorizar la disponibilidad de servicios, identificar interrupciones y utilizar comprobaciones técnicas para investigar su causa.

En el proyecto se ha utilizado Uptime Kuma para comprobar servicios autohospedados. Esta guía presenta un ejemplo reproducible con nombres y destinos genéricos.

Las pruebas de caída y las notificaciones se describen como procedimientos de validación, sin asumir que ya se hayan realizado.

---

## 1. Alcance de la monitorización

La monitorización de disponibilidad permite comprobar si un servicio responde desde un punto concreto de la red.

| Comprobación | Qué permite observar |
|---|---|
| HTTP/HTTPS | Respuesta de una aplicación web |
| TCP | Disponibilidad de un puerto |
| Ping | Respuesta ICMP de un equipo |
| DNS | Resolución de un registro |
| Palabra clave HTTP | Presencia de un contenido esperado en una respuesta |

Una respuesta correcta no garantiza que todas las funciones de la aplicación estén operativas.

Uptime Kuma complementa la administración de servicios. No sustituye a un SIEM ni al análisis de eventos de seguridad.

---

## 2. Preparar el despliegue

**Ejecutar en:** la VM Linux con Docker y Docker Compose, no en el host Proxmox.

```bash
mkdir -p ~/docker/uptime-kuma
cd ~/docker/uptime-kuma
nano compose.yaml
```

Contenido de `compose.yaml`:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    restart: unless-stopped
    ports:
      - "127.0.0.1:3001:3001"
    volumes:
      - uptime_kuma_data:/app/data

volumes:
  uptime_kuma_data:
```

### Explicación

- La aplicación conserva sus datos en `/app/data`.
- El volumen debe utilizar almacenamiento local compatible; no NFS.
- El panel solo se publica en la interfaz local de la VM.
- La etiqueta `2` corresponde a una versión mayor y puede actualizarse. Para reproducir una versión exacta, registrar una etiqueta específica o un digest.

Este ejemplo es para una instalación nueva. Una instalación existente debe actualizarse siguiendo su procedimiento de migración y con una copia previa.

---

## 3. Iniciar y comprobar el servicio

Desde la carpeta del proyecto:

```bash
sudo docker compose config --quiet
sudo docker compose pull
sudo docker compose up -d
```

Comprobar el estado y los registros:

```bash
sudo docker compose ps
sudo docker compose logs --tail=100 uptime-kuma
```

Comprobar la respuesta local:

```bash
curl --output /dev/null \
  --silent \
  --show-error \
  --max-time 10 \
  --write-out 'HTTP: %{http_code}\n' \
  http://127.0.0.1:3001
```

Un código HTTP indica que se ha recibido una respuesta. Su interpretación depende del código y del comportamiento esperado de la aplicación.

---

## 4. Acceder al panel

**Ejecutar en:** el equipo desde el que se abrirá el navegador.

```bash
ssh -N -L 3001:127.0.0.1:3001 usuario@IP_DE_LA_VM
```

Mantener abierta la sesión y acceder a:

```text
http://localhost:3001
```

El tráfico entre el equipo y la VM viaja por el túnel SSH.

En el primer acceso:

1. Crear la cuenta administrativa.
2. Configurar una contraseña robusta.
3. Revisar la zona horaria y las preferencias disponibles.
4. Preparar nombres descriptivos para los monitores.

---

## 5. Crear un monitor HTTP/HTTPS

Desde el panel, seleccionar la opción para añadir un monitor.

Ejemplo de configuración:

| Campo | Valor ilustrativo |
|---|---|
| Tipo | HTTP/HTTPS |
| Nombre | Aplicación de laboratorio |
| URL | `https://servicio.example.com` |
| Intervalo | 60 segundos |
| Reintentos | 2 |
| Código esperado | `200`, si corresponde al endpoint |

El dominio es ficticio y debe sustituirse por un servicio accesible desde Uptime Kuma.

### Procedimiento

1. Introducir el nombre y la URL.
2. Configurar el intervalo.
3. Ajustar los reintentos para evitar alertas por fallos transitorios.
4. Definir los códigos de respuesta aceptados.
5. Guardar.
6. Observar varias comprobaciones consecutivas.

El intervalo y los reintentos influyen en el tiempo necesario para detectar una caída.

### Elegir el destino adecuado

Una página de inicio puede responder aunque una función interna esté fallando.

Cuando la aplicación disponga de un endpoint de salud, valorar su uso. También se puede comprobar una palabra clave o un resultado específico para validar algo más que la conexión HTTP.

No desactivar la validación del certificado TLS como solución habitual a un error: revisar el nombre, la cadena de confianza y la caducidad.

---

## 6. Comprobaciones TCP y ping

### Monitor TCP

Permite comprobar si se puede establecer una conexión con un puerto.

Ejemplo:

| Campo | Valor ilustrativo |
|---|---|
| Tipo | TCP |
| Nombre | SSH de laboratorio |
| Destino | Dirección del equipo de prueba |
| Puerto | `22` |

Que el puerto responda no demuestra que la autenticación o todas las funciones del servicio sean correctas.

### Monitor ping

Permite comprobar la respuesta ICMP de un equipo.

Un equipo puede funcionar y no responder a ping si su firewall bloquea ICMP. El resultado debe interpretarse según su configuración.

**Aprendizaje:** elegir una comprobación que represente el servicio que realmente se quiere vigilar.

---

## 7. Entender desde dónde se comprueba

Los monitores se ejecutan desde Uptime Kuma y utilizan su conectividad.

Dentro de su contenedor:

- `localhost` y `127.0.0.1` hacen referencia al propio contenedor.
- Un nombre de servicio Docker requiere una red compartida y resolución adecuada.
- Una dirección de la red local necesita una ruta y permisos de acceso.
- Un dominio público puede recorrer un proxy o un túnel diferente al acceso interno.

Comprobar un servicio desde el portátil no demuestra que Uptime Kuma pueda acceder a él.

Un monitor interno y otro externo pueden aportar información distinta sobre la misma aplicación.

---

## 8. Investigar una alerta

### Paso 1: revisar el monitor

Anotar:

- Hora del primer fallo.
- Mensaje de error.
- Destino comprobado.
- Última respuesta correcta.
- Otros monitores afectados.

### Paso 2: revisar el entorno de la aplicación

**Ejecutar en:** VM o contenedor que aloja el servicio afectado.

```bash
free -h
df -h
ip -br address
ip route
ss -tuln
```

En sistemas con systemd:

```bash
systemctl --failed
journalctl -b -p warning --no-pager -n 100
```

### Paso 3: revisar la aplicación Docker

Desde la carpeta de la aplicación afectada:

```bash
sudo docker compose ps
sudo docker compose logs --tail=100
```

### Paso 4: comprobar la respuesta HTTP

Desde un equipo con la conectividad que se quiere evaluar:

```bash
curl --output /dev/null \
  --silent \
  --show-error \
  --max-time 10 \
  --write-out 'HTTP: %{http_code}\nTiempo: %{time_total}s\n' \
  https://servicio.example.com
```

Esta prueba solo representa el acceso desde el equipo donde se ejecuta.

### Interpretación inicial

| Resultado | Posible línea de investigación |
|---|---|
| Error de resolución | DNS |
| Conexión rechazada | Puerto incorrecto, servicio detenido o rechazo activo |
| Tiempo de espera agotado | Filtrado, ruta, saturación o servicio sin respuesta |
| Error TLS | Certificado, nombre del destino o confianza |
| HTTP 401/403 | Autenticación o permisos |
| HTTP 500 | Error de la aplicación |
| HTTP 502/504 | Proxy y comunicación con el servicio de destino |

Estos resultados orientan el diagnóstico; no confirman por sí solos una causa.

---

## 9. Prueba controlada de caída y recuperación

Realizar esta prueba únicamente sobre un servicio de laboratorio prescindible. Mantener Uptime Kuma funcionando.

### Procedimiento de validación

1. Comprobar que el monitor está operativo.
2. Anotar la hora.
3. Detener el servicio de prueba.
4. Esperar a que se cumplan los intervalos y reintentos configurados.
5. Comprobar que se registra la caída.
6. Iniciar de nuevo el servicio.
7. Confirmar que el monitor detecta la recuperación.

Para una aplicación gestionada con Compose:

**Ejecutar en:** la carpeta de la aplicación de prueba.

Sustituir `servicio_prueba` por el nombre real del servicio en su archivo Compose.

```bash
sudo docker compose stop servicio_prueba
```

Después de comprobar la detección:

```bash
sudo docker compose start servicio_prueba
```

### Evidencias que registrar

- Captura del estado previo.
- Hora de parada e inicio.
- Evento de caída.
- Evento de recuperación.
- Tiempo aproximado de detección.

Esta prueba permite validar el comportamiento del monitor, en lugar de asumir que funcionará ante una interrupción real.

---

## 10. Notificaciones

Como ampliación, se puede configurar un canal de notificaciones compatible con la instalación.

### Validación del canal

1. Configurar el proveedor y las credenciales necesarias.
2. Enviar una notificación de prueba.
3. Asociar el canal a los monitores correspondientes.
4. Comprobar la recepción durante una caída controlada.
5. Comprobar también la notificación de recuperación.

El botón de prueba valida el envío, pero no sustituye la comprobación de una alerta real del monitor.

No publicar tokens, contraseñas ni direcciones privadas en capturas o archivos del repositorio.

---

## 11. Límites y mantenimiento

- Si Uptime Kuma y el servicio dependen del mismo host, una caída del host puede impedir que se envíe la alerta.
- La monitorización externa permite detectar fallos que un monitor dentro del mismo entorno no puede comunicar.
- Los mantenimientos planificados deben distinguirse de las interrupciones inesperadas.
- Los monitores deben revisarse cuando cambien las direcciones, certificados o servicios.
- La configuración y los datos de Uptime Kuma necesitan respaldo.

---

## Conocimientos practicados

- Despliegue de una herramienta de monitorización.
- Selección de comprobaciones HTTP, TCP e ICMP.
- Interpretación de estados y tiempos de respuesta.
- Identificación del punto desde el que se realiza una comprobación.
- Diagnóstico mediante red, recursos y registros.
- Diseño de pruebas de caída y recuperación.
- Documentación de evidencias.

---

## Referencias

- [Repositorio y documentación de Uptime Kuma](https://github.com/louislam/uptime-kuma)
- [Wiki de Uptime Kuma](https://github.com/louislam/uptime-kuma/wiki)

---

[Volver al inicio](../README.md) · [Anterior: Docker y Portainer](03-docker-portainer.md)
