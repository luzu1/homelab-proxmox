# 07 – Home Assistant y servicios Zigbee/MQTT

## Objetivo

Documentar el despliegue de Home Assistant OS, la integración de dispositivos Zigbee y la comunicación entre servicios mediante MQTT.

El trabajo realizado incluye Home Assistant en una máquina virtual e integración de dispositivos y servicios de domótica.

Los ejemplos siguientes utilizan nombres e identificadores ficticios. Las automatizaciones de ejemplo son propuestas de laboratorio, no evidencias de pruebas ya realizadas.

---

## 1. Componentes y funciones

| Componente | Función |
|---|---|
| Home Assistant OS | Plataforma para integrar dispositivos y automatizaciones |
| Coordinador Zigbee | Comunicación con la red de dispositivos Zigbee |
| Zigbee2MQTT | Traducción entre dispositivos Zigbee y mensajes MQTT |
| Broker MQTT | Recepción y distribución de mensajes |
| Sensores y actuadores | Generación de eventos y ejecución de acciones |

En esta guía, Zigbee2MQTT representa el método de integración utilizado para explicar la comunicación por MQTT.

ZHA es otra opción de integración. No debe utilizar simultáneamente el mismo coordinador que Zigbee2MQTT.

---

## 2. Preparar la máquina virtual

Home Assistant OS se distribuye como una imagen de disco preparada.

### Procedimiento general

1. Descargar la imagen oficial para KVM/Proxmox.
2. Descomprimirla si es necesario.
3. Crear una VM sin ISO de instalación.
4. Configurar firmware UEFI.
5. Preparar el disco EFI sin claves de Secure Boot preinscritas.
6. Asignar recursos adecuados a las integraciones previstas.
7. Seleccionar el puente de red.
8. Importar y conectar el disco de Home Assistant OS.

Como punto de partida para un laboratorio se pueden asignar 2 vCPU y 4 GB de RAM, ajustando después según el consumo.

---

## 3. Importar el disco

**Ejecutar en:** host Proxmox.

Este ejemplo supone:

- Una VM vacía ya creada con identificador `220`.
- Una imagen descomprimida en `/ruta/haos.qcow2`.
- Un almacenamiento llamado `almacenamiento-vm` que admite discos de VM.

```bash
qm importdisk 220 /ruta/haos.qcow2 almacenamiento-vm
```

Los valores deben sustituirse por los del laboratorio.

Después, desde **Hardware** de la VM:

1. Localizar el disco importado, que puede aparecer como disco sin utilizar.
2. Conectarlo mediante un controlador compatible.
3. Revisar el orden de arranque.
4. Seleccionar el disco importado como dispositivo de inicio.
5. Comprobar que no se está intentando arrancar desde un disco vacío.

Consultar la configuración:

```bash
qm config 220
```

Iniciar la VM:

```bash
qm start 220
```

Abrir la consola y observar el arranque.

---

## 4. Primer acceso a Home Assistant

Desde un equipo con acceso a la red del laboratorio:

```text
http://IP_DE_HOME_ASSISTANT:8123
```

Completar el asistente:

1. Crear la cuenta administrativa.
2. Revisar la zona horaria.
3. Configurar las preferencias.
4. Comprobar las integraciones detectadas.
5. Revisar el estado general.

Home Assistant OS se administra con sus herramientas e interfaz. No se trata como una instalación Debian convencional ni se utiliza `apt` para mantenerlo.

### Comprobaciones desde la consola de HAOS

En el indicador `ha >`:

```text
core info
supervisor info
network info
core logs
```

Estas consultas permiten revisar el núcleo, el supervisor, la red y los registros.

---

## 5. Preparar el coordinador Zigbee

El coordinador debe estar accesible desde el entorno que ejecuta Zigbee2MQTT.

**Ejecutar en:** el sistema Linux donde está conectado el coordinador.

```bash
lsusb
ls -l /dev/serial/by-id/
```

La ruta por identificador permite reconocer el adaptador con más estabilidad que un nombre como `/dev/ttyUSB0`.

### Acceso desde un entorno virtualizado

| Entorno | Qué revisar |
|---|---|
| VM | Asignación USB desde Proxmox |
| LXC | Acceso al dispositivo y correspondencia de permisos |
| Docker | Dispositivo expuesto al contenedor de la aplicación |

Si Zigbee2MQTT se ejecuta en un entorno separado de Home Assistant, el coordinador se asigna a ese entorno.

No habilitar privilegios amplios como primera solución a un problema de permisos.

---

## 6. Preparar MQTT

Este apartado supone que ya existe un broker MQTT operativo, como Mosquitto.

Antes de conectar aplicaciones:

- Crear credenciales específicas.
- Restringir el acceso a los clientes previstos.
- Comprobar conectividad con el broker.
- Revisar permisos sobre los temas.
- Utilizar TLS cuando se necesite cifrar la comunicación.

MQTT sin TLS no cifra los mensajes ni las credenciales en tránsito.

### Prueba básica de comunicación

Desde una herramienta cliente MQTT autorizada:

1. Suscribirse al tema `lab/prueba`.
2. Publicar un mensaje de prueba en ese tema.
3. Confirmar su recepción.
4. Comprobar que las credenciales incorrectas son rechazadas.

Esta prueba permite validar el broker antes de incorporar Zigbee2MQTT y Home Assistant.

---

## 7. Configuración de Zigbee2MQTT

El siguiente fragmento muestra los campos principales de `configuration.yaml`. No sustituye la configuración completa de una instalación.

```yaml
mqtt:
  server: mqtt://BROKER_DEL_LABORATORIO:1883
  base_topic: zigbee2mqtt
  user: USUARIO_MQTT
  password: CAMBIAR_POR_UN_SECRETO

serial:
  port: /dev/serial/by-id/IDENTIFICADOR_DEL_COORDINADOR
  adapter: ember

homeassistant:
  enabled: true
```

### Qué adaptar

- Dirección y puerto del broker.
- Credenciales.
- Ruta visible desde el entorno de Zigbee2MQTT.
- Tipo de adaptador.

`ember` es un ejemplo para coordinadores compatibles. El valor debe elegirse según el modelo y el firmware; no todos utilizan el mismo controlador.

Si el adaptador se expone dentro de Docker con otra ruta, la configuración debe utilizar la ruta interna.

### Comprobar el servicio

Si se administra mediante Docker Compose, desde su carpeta:

```bash
sudo docker compose ps
sudo docker compose logs --tail=100
```

Buscar errores relacionados con:

- Acceso al puerto serie.
- Adaptador ocupado.
- Firmware o controlador.
- Autenticación MQTT.
- Resolución del nombre del broker.

---

## 8. Integrar MQTT en Home Assistant

Desde **Ajustes → Dispositivos y servicios**:

1. Añadir la integración MQTT.
2. Introducir el broker y las credenciales.
3. Confirmar la conexión.
4. Comprobar la aparición de dispositivos mediante descubrimiento MQTT.

El broker actúa como punto común de comunicación entre Home Assistant y Zigbee2MQTT.

Si un dispositivo aparece en Zigbee2MQTT pero no en Home Assistant, revisar la conexión MQTT y la configuración de descubrimiento.

---

## 9. Emparejar y comprobar dispositivos

Desde Zigbee2MQTT:

1. Habilitar temporalmente la incorporación de dispositivos.
2. Poner el dispositivo en modo de emparejamiento.
3. Esperar a que termine su identificación.
4. Asignar un nombre descriptivo.
5. Cerrar la incorporación de nuevos dispositivos.

### Validación funcional

| Dispositivo | Prueba |
|---|---|
| Sensor de apertura | Abrir y cerrar, comprobando ambos estados |
| Sensor de movimiento | Generar movimiento y esperar a su restablecimiento |
| Sensor de temperatura | Revisar recepción y actualización de valores |
| Enchufe | Encender y apagar una carga de prueba adecuada |

Comprobar el evento tanto en Zigbee2MQTT como en Home Assistant permite localizar dónde se interrumpe la comunicación.

Los dispositivos a batería pueden actualizar determinados valores con menor frecuencia.

---

## 10. Automatización de laboratorio

Ejemplo: registrar una notificación interna al abrir un sensor de prueba.

En el editor YAML de una nueva automatización de Home Assistant:

```yaml
alias: "Laboratorio - Apertura detectada"
description: "Comprobación del evento de un sensor de apertura"
triggers:
  - trigger: state
    entity_id: binary_sensor.apertura_laboratorio
    from: "off"
    to: "on"
conditions: []
actions:
  - action: persistent_notification.create
    data:
      title: "Prueba de sensor"
      message: "El sensor de laboratorio ha detectado una apertura."
mode: single
```

Sustituir la entidad por la del dispositivo real y comprobar que sus estados corresponden a `off` y `on`.

### Comprobación

1. Guardar la automatización.
2. Abrir el sensor físicamente.
3. Comprobar la notificación.
4. Revisar la traza de ejecución.

Ejecutar manualmente las acciones no valida que el disparador físico funcione.

---

## 11. Diagnóstico por capas

| Síntoma | Qué revisar |
|---|---|
| Home Assistant no inicia | Consola de la VM, disco y orden de arranque |
| La web no responde | Estado de HAOS, dirección y conectividad |
| Zigbee2MQTT no abre el adaptador | Asignación del dispositivo, permisos y procesos que lo utilizan |
| No conecta al broker | Dirección, puerto, credenciales y registros |
| Un sensor no actualiza | Batería, cobertura, emparejamiento y frecuencia de reporte |
| El evento llega a Zigbee2MQTT pero no a Home Assistant | MQTT y descubrimiento |
| El estado cambia pero la automatización no actúa | Entidad, disparador, condiciones y trazas |

**Aprendizaje:** investigar cada componente antes de reiniciar todo el sistema.

---

## 12. Copias de seguridad

La recuperación debe contemplar:

- Home Assistant y su configuración.
- Datos y configuración de Zigbee2MQTT.
- Respaldo del coordinador cuando sea compatible.
- Configuración, usuarios y permisos del broker.
- Credenciales necesarias para restaurar las conexiones.

Un backup de la VM de Home Assistant no incluye automáticamente servicios alojados fuera de ella.

Las claves Zigbee y los secretos MQTT forman parte de la información privada del entorno y no se publican en el repositorio.

---

## Conocimientos practicados

- Despliegue de una aplicación mediante imagen de disco.
- Administración de Home Assistant OS.
- Acceso a dispositivos desde entornos virtualizados.
- Comunicación mediante MQTT.
- Integración y comprobación de sensores.
- Diseño de automatizaciones basadas en eventos.
- Diagnóstico por componentes.
- Identificación de los elementos necesarios para recuperar el servicio.

---

## Referencias

- [Home Assistant OS en máquinas virtuales](https://www.home-assistant.io/installation/alternative/)
- [Integración MQTT de Home Assistant](https://www.home-assistant.io/integrations/mqtt/)
- [Configuración MQTT de Zigbee2MQTT](https://www.zigbee2mqtt.io/guide/configuration/mqtt.html)
- [Adaptadores de Zigbee2MQTT](https://www.zigbee2mqtt.io/guide/configuration/adapter-settings.html)

---

[Volver al inicio](../README.md) · [Anterior: Immich y migración](06-immich-migracion.md)
