# Detector de Presencia IoT

Un detector de presencia IoT compacto y de bajo consumo que informa ocupación y telemetría básica (batería, intensidad de señal, tiempo de actividad) a un sistema de domótica o a un broker MQTT. Este proyecto está pensado para habitaciones pequeñas, pasillos o puertas donde la información de presencia es útil para automatizaciones, registro o ahorro de energía.

> NOTA: Este README es un borrador listo para editar. Sustituye los marcadores de posición (por ejemplo, DEVICE_NAME, MQTT_BROKER) y las secciones marcadas con TODO por valores que correspondan a tu hardware y despliegue.

## Contenido
- Descripción del proyecto
- Funcionalidades
- Hardware
- Conexiones
- Opciones de firmware
- Configuración de ejemplo (ESPHome + MQTT)
- Temas MQTT y payloads
- Recomendaciones de alimentación / batería
- Instalación y despliegue
- Solución de problemas
- Contribuciones
- Licencia
- Contacto

## Descripción del proyecto
Detector de Presencia IoT lee un sensor de movimiento/presencia y publica eventos de presencia y telemetría a tu stack de domótica. Está diseñado para bajo consumo y fácil integración con sistemas como Home Assistant, Node-RED o cualquier servicio compatible con MQTT.

Usos típicos:
- Encender/apagar luces automáticamente
- Registrar uso de habitaciones a lo largo del tiempo
- Disparar notificaciones o automatizaciones

## Funcionalidades
- Detección de presencia/movimiento (PIR / radar / otro sensor)
- Publica estado de presencia (ocupado / desocupado) y telemetría:
  - nivel de batería (si es alimentado por batería)
  - intensidad de señal Wi‑Fi (RSSI)
  - uptime / último visto
- Modos de bajo consumo (depende del hardware/firmware)
- Integración opcional con Home Assistant (ESPHome o MQTT)
- Actualizaciones OTA (si se usa ESPHome u otro firmware que lo soporte)

## Hardware
Componentes típicos:
- Microcontrolador: ESP8266, ESP32 u otro MCU con Wi‑Fi
- Sensor de presencia: PIR (ej. HC‑SR501) o sensor de microondas/radar
- Alimentación: USB, batería LiPo + cargador (TP4056), o pila botón (según componentes)
- Opcional: divisor de tensión o IC medidor de batería para lectura de voltaje

Ejemplo:
- ESP8266 NodeMCU o WeMos D1 mini
- Sensor PIR HC‑SR501
- Batería LiPo 3.7V + módulo de carga TP4056
- Resistencias para divisor de tensión si mides batería por ADC

## Conexiones (Wiring)
Ejemplo sencillo para ESP8266 + PIR:
- PIR VCC -> 3.3V (o 5V según el sensor; revisar especificaciones)
- PIR GND -> GND
- PIR OUT -> GPIO (ej. D2 / GPIO4)
- Medida de batería (opcional) -> pin ADC a través de divisor de tensión

Importante: Confirma la tensión de alimentación del sensor (algunos requieren 5V).

Si usas deep sleep, asegúrate de que el pin usado para despertar sea compatible con tu MCU y la conexión permita el modo.

## Opciones de firmware
Puedes implementar el firmware de distintas maneras. Dos opciones comunes:

1. ESPHome (recomendado para usuarios de Home Assistant)
   - Configuración rápida con YAML
   - Integración MQTT o nativa con Home Assistant
   - Actualizaciones OTA

2. Arduino / PlatformIO (sketch en C++)
   - Control total sobre gestión de energía y lógica personalizada
   - Útil para técnicas avanzadas de bajo consumo o protocolos personalizados

A continuación un ejemplo de configuración ESPHome de inicio.

### Ejemplo de configuración ESPHome (reemplaza los marcadores)
```yaml
esphome:
  name: presence_detector_DEVICE_NAME
  platform: ESP8266
  board: nodemcuv2

wifi:
  ssid: "TU_SSID"
  password: "TU_CONTRASEÑA_WIFI"
  # Opcional: IP estática / gateway si es necesario

mqtt:
  broker: "MQTT_BROKER"
  username: "MQTT_USER"
  password: "MQTT_PASSWORD"

sensor:
  - platform: wifi_signal
    name: "Presencia Detector RSSI WiFi"
    update_interval: 60s

binary_sensor:
  - platform: gpio
    pin:
      number: D2
      mode: INPUT
      inverted: false
    name: "Detector de Presencia - Movimiento"
    device_class: motion
    on_press:
      - mqtt.publish:
          topic: presence/DEVICE_NAME/event
          payload: "motion_detected"

# Medición de batería (ejemplo ADC)
# sensor:
#   - platform: adc
#     pin: A0
#     name: "Batería Detector de Presencia"
#     update_interval: 300s
#     filters:
#       - multiply: 2.0  # ajustar según divisor de tensión

# Deep sleep (nota: diferencias entre ESP8266 y ESP32)
# deep_sleep:
#   run_duration: 10s
#   sleep_duration: 60s
```

## Temas MQTT y payloads
La estructura de topics es flexible. Recomendación simple:

- presence/DEVICE_NAME/state
  - payload: "on" / "off" o "ocupado" / "libre"
- presence/DEVICE_NAME/event
  - payload: "motion_detected" o JSON para eventos más ricos
- presence/DEVICE_NAME/telemetry/battery
  - payload: porcentaje o voltaje numérico
- presence/DEVICE_NAME/telemetry/rssi
  - payload: valor numérico (dBm)
- presence/DEVICE_NAME/telemetry/uptime
  - payload: segundos

Ejemplo de payload JSON:
{
  "presence": "on",
  "battery": 83,
  "rssi": -62,
  "uptime": 3600
}

Ajusta los nombres de topics para encajar con tu convención o con el descubrimiento de Home Assistant.

## Recomendaciones de alimentación / batería
- Para funcionamiento con batería, usa regulación y gestión adecuadas (carga/protección).
- Los sensores PIR y el radio Wi‑Fi pueden provocar picos de corriente: elige baterías y reguladores que soporten esos picos.
- Considera usar deep sleep o modos de bajo consumo entre lecturas. ESP32/ESP8266 gestionan el deep sleep de forma distinta; consulta su documentación.
- Para mayor duración de batería:
  - Minimiza el tiempo Awake y la frecuencia de publicación
  - Considera sensores radar con mejor sensibilidad permitiendo menos tiempo activo
  - Reduce telemetría innecesaria

## Instalación y despliegue
1. Elige firmware (ESPHome o Arduino) y edita la configuración:
   - Credenciales Wi‑Fi
   - Broker MQTT (o integración nativa)
   - Pines GPIO y escalado de ADC para tu hardware
2. Flashea el dispositivo:
   - ESPHome: compila y sube por USB o OTA (la primera vez por USB)
   - Arduino/PlatformIO: compila y flashea con tu toolchain
3. Valida mensajes:
   - Conéctate al broker MQTT y suscríbete a los topics usados por el dispositivo
   - Verifica que lleguen telemetría y eventos de presencia
4. Montaje y ubicación:
   - Evita apuntar un PIR directamente a rejillas de ventilación, fuentes de calor o luz solar directa

## Solución de problemas
- No llegan mensajes MQTT:
  - Verifica credenciales Wi‑Fi y dirección del broker
  - Comprueba credenciales MQTT y logs del broker
- Falsos positivos/negativos:
  - Ajusta la colocación del sensor y sensibilidad (si dispone)
  - Implementa debounce o exige múltiples disparos para marcar "ocupado"
- Consumo elevado de batería:
  - Asegúrate de que el deep sleep está activado y que el hardware puede despertarse correctamente
  - Reduce frecuencia de publicaciones y telemetría
- Imposible actualizar vía OTA:
  - Flashea por USB y confirma la conectividad de red

## Contribuciones
Se aceptan contribuciones, informes de errores y solicitudes de funciones. Por favor:
- Abre un issue describiendo el problema o la función deseada
- Envía un pull request con una descripción clara y pruebas o actualizaciones de documentación según sea necesario

Si quieres colaborar en características específicas (modos de bajo consumo, soporte radar, descubrimiento en Home Assistant), abre un issue o empieza una discusión.

## Licencia
Especifica una licencia para tu proyecto. Ejemplo:
- MIT License — ver archivo LICENSE

## Agradecimientos
- ESPHome, Home Assistant y autores de las librerías y hardware usados en este proyecto.

## Contacto
Mantenedor: stradcastle  
Repositorio del proyecto: https://github.com/stradcastle/presence_detector_iot

---
TODO: Si quieres, puedo:
- Adaptar este README a tu hardware exacto (indica MCU y modelo de sensor),
- Generar un YAML de ESPHome listo para flashear con tus credenciales y pines,
- Añadir diagramas o imágenes de conexión,
- O crear un sketch de Arduino/PlatformIO de ejemplo.
Indícame qué prefieres y los detalles del hardware.
