# Declaración de fuentes y uso de IA

Proyecto: Sensor de ocupación de estacionamiento IoT (TEI201, UAI).

Este archivo declara todas las librerías, el código externo y el uso de herramientas de inteligencia artificial en el desarrollo del proyecto.

## Librerías utilizadas

| Librería | Versión | Uso en el proyecto | Fuente |
|---|---|---|---|
| WiFi.h | Incluida en el core ESP32 de Arduino (v3.x) | Conexión a la red WiFi (modo estación) y creación de red propia (modo punto de acceso) | https://github.com/espressif/arduino-esp32/tree/master/libraries/WiFi |
| WebServer.h | Incluida en el core ESP32 de Arduino (v3.x) | Servidor HTTP que publica la página web del dashboard y el endpoint JSON `/estado` | https://github.com/espressif/arduino-esp32/tree/master/libraries/WebServer |
| Core Arduino ESP32 (Espressif) | v3.x | Funciones base: `pulseIn()`, `tone()`, `digitalWrite()`, `millis()`, `Serial` | https://github.com/espressif/arduino-esp32 |

No se instalaron librerías externas adicionales.

## Código externo adaptado

### Conexión WiFi y servidor web (`conectarWifi`, `handleRaiz`, `handleEstado` en `firmware/sensor_estacionamiento.ino`)

- Fuente: patrón de uso estándar de los ejemplos oficiales del core ESP32 (`WiFi/WiFiClient` y `WebServer/HelloServer`), https://github.com/espressif/arduino-esp32/tree/master/libraries
- Adaptación: se agregó un tiempo límite de 10 s para la conexión; si no se logra, el ESP32 crea su propia red WiFi ("Estacionamiento-A1", página en 192.168.4.1), para que la demo no dependa de una red externa. El endpoint `/estado` entrega en JSON el estado, la distancia, el porcentaje de respuesta del sensor, los cambios de estado y el tiempo encendido.

### Medición con sensor ultrasónico (`medirUnaVez` en `firmware/sensor_estacionamiento.ino`)

- Fuente: método estándar de lectura Trig/Echo para sensores tipo HC-SR04 (pulso de disparo, `pulseIn()` y conversión con la velocidad del sonido, 343 m/s), documentado en la hoja de datos del sensor y en la referencia de Arduino: https://www.arduino.cc/reference/en/language/functions/advanced-io/pulsein/
- Adaptación: el pulso de disparo se alargó a 20 µs porque el SR04M-2 no respondía bien con 10 µs, y el tiempo de espera se amplió a 70 ms para reconocer el pulso de "nada al frente" (~55 ms) que entrega este modelo.

## Uso de Inteligencia Artificial

### Firmware completo (`firmware/sensor_estacionamiento.ino`)

- Herramienta: Claude (Anthropic), octubre y noviembre de 2026.
- Uso: generación de la estructura del programa y de las funciones de medición, filtrado, lógica de estado, actuadores, conectividad y página web. El código se desarrolló en 8 versiones, corrigiendo problemas detectados en las pruebas físicas del prototipo.
- Prompt inicial: "Escribe el código para un ESP32 con un sensor ultrasónico SR04M-2 que detecte si un estacionamiento está libre u ocupado, muestre el estado con un LED bicolor y un buzzer, y lo publique en una página web".
- Adaptación hecha por el equipo a partir de las pruebas (ver `testing/`):
  - Se cambió la alimentación del sensor de 5 V a 3,3 V y se dejó una resistencia de 1 kΩ en serie en Echo, porque con el divisor de voltaje el sensor no respondía.
  - Se reemplazó el buzzer activo (defectuoso) por uno pasivo, por lo que el código pasó a usar `tone()` a 2 kHz.
  - Se descartan lecturas bajo 20 cm, porque en la zona ciega aparecían valores falsos de 1,8 a 3,1 cm.
  - Los ciclos en que el sensor no responde dejaron de contarse como "libre", porque provocaban cambios de estado erróneos.
  - Se ajustaron las confirmaciones necesarias (de 3 a 2) y el descanso entre disparos (de 60 a 100 ms), con lo que el cambio de estado bajó a ~2 s.
  - Los coeficientes de calibración (pendiente 1,011 y offset 4,0 cm) salen de una regresión lineal hecha por el equipo con 4 distancias medidas con huincha (R² = 0,99997); se validó fuera de muestra a 80 cm con error de ~0,8 cm.
  - Los umbrales de 60 cm (ocupado) y 80 cm (libre) se definieron para la demo sobre una mesa.
- Comprensión: el equipo entiende que cada ciclo toma 5 lecturas y usa la mediana para eliminar valores extremos; que la histéresis (dos umbrales distintos) evita que el estado parpadee cuando el objeto está justo en el límite; que se exigen 2 mediciones seguidas para cambiar de estado, lo que filtra objetos que pasan (por ejemplo, un peatón); y que la página web consulta `/estado` cada segundo con `fetch()` para actualizarse sin recargar. Cada cambio se probó en el prototipo y se verificó en el Monitor Serie.

### Diagnóstico de fallas del prototipo

- Herramienta: Claude (Anthropic), octubre y noviembre de 2026.
- Uso: apoyo para interpretar las lecturas del Monitor Serie y proponer hipótesis ante fallas (sensor sin respuesta con el divisor de voltaje, buzzer que no sonaba, falso contacto de la sonda dentro del encapsulado). Las pruebas físicas, las mediciones y las decisiones finales las hizo el equipo.
