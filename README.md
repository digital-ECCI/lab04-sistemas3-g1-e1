# Lab05: MQTT en Raspberry Pi + Microcontrolador con MicroPython

**Curso:** Digitales III — Universidad ECCI  
**Fecha de realización:** 17 de abril de 2026

---

## 1. Procedimiento llevado a cabo

### 1.1 Instalación y configuración de Mosquitto en la Raspberry Pi

Se actualizó el sistema operativo de la Raspberry Pi y se instaló el broker MQTT **Mosquitto** junto con sus herramientas de cliente:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install mosquitto mosquitto-clients -y
```

A continuación se habilitó e inició el servicio para que arranque automáticamente con el sistema:

```bash
sudo systemctl enable mosquitto
sudo systemctl start mosquitto
sudo systemctl status mosquitto
```

Se verificó que el servicio se encontrara en estado `active (running)`, confirmando que el broker estaba operativo en el puerto **1883** (por defecto).

---

### 1.2 Configuración de Node-RED con nodos MQTT

Desde la interfaz web de Node-RED (accedida vía SSH + IP de la Raspberry Pi, puerto `1880`), se creó el flujo **"Control LED MQTT"** con los siguientes nodos:

| Nodo | Tipo | Función |
|------|------|---------|
| `Encender LED` | `inject` | Publica el payload `"ON"` |
| `Apagar LED` | `inject` | Publica el payload `"OFF"` |
| `Publicar LED` | `mqtt out` | Publica en el topic `micro/led/control` |
| `Estado LED` | `mqtt in` | Se suscribe al topic `micro/led/control` |
| `Estado` | `text` (dashboard) | Muestra el estado actual del LED |

La conexión con el broker se configuró apuntando a `localhost` en el puerto `1883`, dado que Mosquitto corre en la misma Raspberry Pi que Node-RED.

Adicionalmente se implementó un **dashboard** (Node-RED Dashboard) con:
- Dos botones: **ON** (verde) y **OFF** (rojo).
- Un campo de texto que muestra el estado actual recibido por el nodo `mqtt in`.

---

### 1.3 Programación del microcontrolador (ESP32 / Raspberry Pi Pico W) con MicroPython

El microcontrolador fue conectado al PC mediante USB con el botón **BOOTSEL** presionado (para Pico W) y programado desde **Thonny IDE** con el intérprete **MicroPython**.

El script cargado en el microcontrolador realiza las siguientes acciones:

1. Se conecta a la red WiFi local.
2. Establece conexión con el broker MQTT en la IP de la Raspberry Pi, puerto `1883`.
3. Se suscribe al topic `micro/led/control`.
4. En el callback de recepción de mensajes:
   - Si recibe `"ON"` → enciende el LED conectado al GPIO.
   - Si recibe `"OFF"` → apaga el LED.

```python
import network
import time
from umqtt.simple import MQTTClient
from machine import Pin

# Configuración WiFi
SSID = "nombre_red"
PASSWORD = "contraseña_red"

# Configuración MQTT
BROKER_IP = "10.177.114.228"  # IP de la Raspberry Pi
TOPIC = b"micro/led/control"

# LED
led = Pin(2, Pin.OUT)  # GPIO2 para ESP32 / ajustar para Pico W

def conectar_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(SSID, PASSWORD)
    while not wlan.isconnected():
        time.sleep(0.5)

def callback(topic, msg):
    if msg == b"ON":
        led.value(1)
    elif msg == b"OFF":
        led.value(0)

conectar_wifi()
client = MQTTClient("pico_client", BROKER_IP)
client.set_callback(callback)
client.connect()
client.subscribe(TOPIC)

while True:
    client.check_msg()
    time.sleep(0.1)
```

> **Nota:** El LED físico fue conectado al GPIO con una resistencia de 220 Ω en serie para limitar la corriente.

---

## 2. Resultados obtenidos

- El broker **Mosquitto** quedó operativo y aceptando conexiones desde múltiples clientes en la red local.
- El flujo de Node-RED se desplegó correctamente; ambos nodos `mqtt in` y `mqtt out` mostraron el indicador **"connected"** en el editor.
- Al presionar el botón **ON** en el dashboard, el LED físico del microcontrolador se encendió de forma inmediata; al presionar **OFF**, se apagó.
- El campo de estado del dashboard reflejó en tiempo real el último mensaje publicado en el topic `micro/led/control`.
- La arquitectura completa (Node-RED → Broker Mosquitto → Microcontrolador) funcionó de extremo a extremo sobre la red WiFi local, con la Raspberry Pi como nodo maestro y el microcontrolador como nodo esclavo.

---

## 3. Evidencia fotográfica

| Descripción | Imagen |
|-------------|--------|
| Setup físico: laptop con Thonny, microcontrolador en protoboard conectado por USB | `foto_setup.jpeg` |
| Microcontrolador (ESP32/Pico W) en protoboard con LED encendido (luz azul onboard visible) | `foto_microcontrolador.jpeg` |
| Flujo Node-RED "Control LED MQTT" y dashboard con botones ON/OFF | `foto_nodered_dashboard.jpeg` |

---

## 4. Archivos entregados

| Archivo | Descripción |
|---------|-------------|
| `flows_lab05.json` | Flujo Node-RED exportado (Control LED MQTT) |
| `main.py` | Script MicroPython para el microcontrolador |
| `README.md` | Este documento de documentación técnica |
