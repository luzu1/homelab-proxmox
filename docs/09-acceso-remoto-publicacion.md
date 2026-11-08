# 09 – Acceso remoto y publicación de servicios

## Objetivo

Documentar los métodos utilizados para acceder a servicios de forma privada y publicar aplicaciones web, junto con sus comprobaciones de funcionamiento.

El proyecto ha incluido trabajo con Tailscale, Cloudflare Tunnel y Nginx Proxy Manager. Los ejemplos siguientes describen procedimientos generales y no muestran la topología ni los dominios de la infraestructura personal.

La validación completa del acceso administrativo remoto al host Proxmox queda pendiente de documentar.

---

## 1. Función de cada herramienta

| Herramienta | Función |
|---|---|
| Tailscale | Conectividad privada entre dispositivos autorizados |
| Cloudflare Tunnel | Conexión entre Cloudflare y servicios de origen mediante `cloudflared` |
| Nginx Proxy Manager | Enrutamiento de peticiones web hacia aplicaciones mediante un proxy inverso |
| SSH | Administración remota y creación de túneles |
| DNS | Resolución de nombres hacia los destinos configurados |

Estas herramientas pueden utilizarse por separado o combinarse según el objetivo.

Un proxy inverso no crea por sí solo una red privada. Un túnel tampoco sustituye automáticamente la autenticación de la aplicación.

---

## 2. Separar administración y acceso a aplicaciones

Antes de configurar un acceso, definir:

- Quién necesita utilizarlo.
- Desde qué dispositivos.
- Qué servicio debe estar disponible.
- Cómo se autentica el usuario.
- Cómo se revoca el acceso.

| Caso | Enfoque del laboratorio |
|---|---|
| Administración por SSH | Acceso privado autorizado |
| Paneles de infraestructura | Acceso privado |
| Aplicación web para usuarios | Publicación con autenticación adecuada |
| Comprobación puntual de un panel local | Túnel SSH |
| Varios servicios web por nombre | Proxy inverso cuando resulte necesario |

La publicación de una aplicación no requiere publicar también sus paneles administrativos.

---

## 3. Comprobar Tailscale

Este procedimiento supone que Tailscale está instalado y que el dispositivo se ha autorizado en la red privada.

**Ejecutar en:** el equipo Linux con Tailscale.

```bash
tailscale status
tailscale ip -4
tailscale netcheck
```

### Qué revisar

- Estado de conexión.
- Identidad del dispositivo.
- Dirección asignada.
- Disponibilidad de otros equipos.
- Condiciones de conectividad.

Las salidas pueden mostrar nombres y direcciones privadas. Deben revisarse antes de publicarlas.

### Comprobar un destino

Sustituir `NOMBRE_DEL_EQUIPO` por un destino autorizado:

```bash
tailscale ping NOMBRE_DEL_EQUIPO
```

Una respuesta confirma conectividad de Tailscale, pero no demuestra que un servicio concreto esté escuchando ni que el usuario tenga permiso para utilizarlo.

---

## 4. Acceso SSH mediante la red privada

**Ejecutar en:** el equipo desde el que se administrará el servidor.

```bash
ssh usuario@DIRECCION_TAILSCALE
```

Este ejemplo utiliza SSH convencional sobre la conectividad de Tailscale. No implica haber activado la función específica Tailscale SSH.

Para que funcione:

- El destino debe ser accesible.
- El servidor SSH debe estar activo.
- La política de acceso debe permitir la conexión.
- El usuario debe superar la autenticación.

En el primer acceso, verificar la identidad del servidor antes de aceptar su clave.

### Si la conexión falla

**Ejecutar en:** la consola del servidor Linux.

```bash
sudo ss -ltnp 'sport = :22'
```

En sistemas Debian o Ubuntu con el servicio `ssh`:

```bash
sudo systemctl status ssh --no-pager
sudo journalctl -u ssh --since "30 minutes ago" --no-pager
```

---

## 5. Abrir un panel local mediante túnel SSH

Un panel puede escuchar únicamente en la interfaz local del servidor y utilizarse mediante un túnel.

Ejemplo para un servicio HTTPS en el puerto `9443`:

**Ejecutar en:** el equipo cliente.

```bash
ssh -N \
  -L 9443:127.0.0.1:9443 \
  usuario@DIRECCION_TAILSCALE
```

Abrir en el navegador:

```text
https://localhost:9443
```

El destino `127.0.0.1:9443` se interpreta desde el servidor SSH.

Si la aplicación está en otro equipo o en otro espacio de red, se necesita un destino accesible desde ese servidor.

Cerrar la sesión SSH termina el túnel.

---

## 6. Publicar una aplicación mediante Cloudflare Tunnel

Este ejemplo utiliza un túnel administrado desde el panel de Cloudflare.

### Procedimiento general

1. Elegir una aplicación de laboratorio.
2. Comprobar que responde desde el entorno donde se ejecutará `cloudflared`.
3. Crear el túnel en Cloudflare.
4. Instalar y registrar el conector siguiendo las instrucciones del panel.
5. Confirmar que el conector aparece conectado.
6. Configurar el nombre público y la dirección del servicio de origen.
7. Definir la autenticación necesaria.
8. Comprobar el acceso desde un cliente externo.

El token del conector es un secreto. No debe aparecer en capturas, comandos publicados ni archivos del repositorio.

### Ejemplo de correspondencia

| Elemento | Valor ilustrativo |
|---|---|
| Nombre público | `app.example.com` |
| Servicio de origen | `http://SERVIDOR_APLICACION:8080` |

La dirección de origen debe ser accesible desde `cloudflared`.

Si el conector se ejecuta en Docker, `localhost` identifica su propio contenedor, no otro contenedor ni el host.

---

## 7. Revisar el conector

### Instalación como servicio Linux

**Ejecutar en:** el sistema que aloja `cloudflared`.

```bash
sudo systemctl status cloudflared --no-pager

sudo journalctl -u cloudflared \
  --since "30 minutes ago" \
  --no-pager
```

### Instalación mediante Docker Compose

Desde la carpeta del proyecto:

```bash
sudo docker compose ps
sudo docker compose logs --tail=100
```

Comprobar:

- Conexión del conector.
- Errores de resolución.
- Fallos de conexión con el origen.
- Errores de certificados.
- Reinicios repetidos.

Que el túnel esté conectado no garantiza que la aplicación de origen responda.

---

## 8. Configurar un proxy inverso con Nginx Proxy Manager

Este apartado supone que Nginx Proxy Manager ya está desplegado y que su panel administrativo es accesible de forma privada.

Desde **Proxy Hosts**, crear una entrada:

| Campo | Valor ilustrativo |
|---|---|
| Domain Names | `app.example.com` |
| Scheme | `http` o `https`, según el origen |
| Forward Hostname / IP | Dirección accesible de la aplicación |
| Forward Port | Puerto de la aplicación |

### Comprobaciones

1. Confirmar que Nginx Proxy Manager alcanza el origen.
2. Introducir el nombre y el destino.
3. Activar WebSockets si la aplicación los necesita.
4. Configurar TLS cuando corresponda.
5. Guardar.
6. Comprobar la respuesta utilizando el nombre previsto.

El panel de administración y los puertos que sirven las aplicaciones cumplen funciones diferentes. El panel no debe convertirse en el destino de una publicación accidental.

### Aplicaciones detrás de un proxy

Algunas aplicaciones requieren configurar:

- URL externa.
- Proxies de confianza.
- Cabeceras reenviadas.
- Cookies y redirecciones.
- WebSockets.

Estos ajustes deben seguir la documentación de cada aplicación.

---

## 9. Combinar Cloudflare Tunnel y el proxy

Cloudflare Tunnel puede dirigir las peticiones directamente a una aplicación.

Si se utiliza Nginx Proxy Manager como intermediario:

1. El túnel debe alcanzar el puerto web del proxy.
2. El proxy debe reconocer el nombre solicitado.
3. La cabecera `Host` debe corresponder a la entrada configurada.
4. El proxy debe alcanzar la aplicación.
5. Los esquemas HTTP/HTTPS deben ser coherentes en cada tramo.

No es obligatorio introducir un proxy adicional si el túnel ya cubre las necesidades del servicio.

### Certificados y HTTPS

El certificado que ve el navegador no demuestra por sí solo que todos los tramos internos estén cifrados.

Cuando un tramo utilice HTTPS, revisar el nombre del certificado y su confianza. Evitar desactivar su validación como solución permanente.

---

## 10. Autenticación y permisos

Publicar un nombre mediante un túnel no restringe automáticamente quién puede acceder a él.

Según el servicio, utilizar:

- Autenticación propia de la aplicación.
- Políticas de Cloudflare Access.
- Restricciones de conectividad privada.
- Cuentas y permisos limitados.

Las políticas de acceso deben probarse con:

1. Un usuario autorizado.
2. Un usuario no autorizado.
3. Una sesión sin autenticar.

Una configuración de Cloudflare Access puede afectar a aplicaciones móviles, APIs e integraciones. Verificar esos clientes antes de dar el acceso por terminado.

En Tailscale también se debe revisar qué usuarios y dispositivos pueden alcanzar cada servicio.

---

## 11. Diagnóstico por tramos

### Comprobar la aplicación de origen

**Ejecutar en:** un entorno con acceso al origen.

```bash
curl --output /dev/null \
  --silent \
  --show-error \
  --max-time 10 \
  --write-out 'HTTP: %{http_code}\nTiempo: %{time_total}s\n' \
  http://SERVIDOR_APLICACION:8080
```

### Comprobar un proxy HTTP por nombre

Sustituir `IP_DEL_PROXY` por su dirección:

```bash
curl --output /dev/null \
  --silent \
  --show-error \
  --max-time 10 \
  --header 'Host: app.example.com' \
  --write-out 'HTTP: %{http_code}\n' \
  http://IP_DEL_PROXY
```

Esta prueba solo comprueba el tramo HTTP del proxy; no valida el certificado ni el acceso público.

### Comprobar el nombre publicado

**Ejecutar en:** un cliente externo.

```bash
getent hosts app.example.com

curl --output /dev/null \
  --silent \
  --show-error \
  --max-time 10 \
  --write-out 'HTTP: %{http_code}\nTiempo: %{time_total}s\n' \
  https://app.example.com
```

Una redirección hacia autenticación puede ser correcta. El resultado se interpreta según la política configurada.

### Orientación del diagnóstico

| Síntoma | Qué revisar |
|---|---|
| No hay acceso privado | Estado de Tailscale, políticas y servicio de destino |
| El nombre público no resuelve | DNS |
| El conector está desconectado | Servicio, credenciales y conectividad saliente |
| El túnel conecta pero la aplicación falla | Dirección y puerto del origen |
| El proxy muestra otra página | Nombre solicitado y configuración del host |
| Hay redirecciones repetidas | Esquemas, URL externa y cabeceras |
| Falla solo un cliente móvil o API | Autenticación y compatibilidad de políticas |
| Error TLS | Nombre, vigencia y confianza del certificado |


---

## Conocimientos practicados

- Diferenciación entre acceso privado y publicación web.
- Comprobación de conectividad con Tailscale.
- Administración mediante SSH.
- Creación de túneles locales.
- Revisión de conectores Cloudflare Tunnel.
- Configuración de proxies inversos.
- Diagnóstico de DNS, HTTP y TLS.
- Validación de autenticación y permisos.

---

## Referencias

- [Comandos de Tailscale](https://tailscale.com/kb/1080/cli)
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/access-controls/)
- [Nginx Proxy Manager](https://nginxproxymanager.com/guide/)

---

[Volver al inicio](../README.md) · [Anterior: diagnóstico y mantenimiento](08-diagnostico-mantenimiento.md)
