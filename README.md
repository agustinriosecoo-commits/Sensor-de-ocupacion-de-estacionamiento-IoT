# Sensor-de-ocupacion-de-estacionamiento-IoT

Sistema IoT que detecta si un espacio de estacionamiento está libre u ocupado mediante un sensor ultrasónico, muestra el estado con un LED y un buzzer, y lo publica en tiempo real en una página web accesible desde el celular. Proyecto del curso TEI201 (Taller de Diseño en Ingeniería), Universidad Adolfo Ibáñez, Grupo 10.

## Integrantes

- Pablo Rojas
- Felipe Blanco
- Agustín Rioseco

Profesor: Camilo Vásquez

## Problema que resuelve

En el corredor Alameda, Estación Central (Santiago), la densificación y el aumento del parque vehicular hacen que los conductores circulen buscando estacionamiento sin saber dónde hay espacios disponibles, lo que genera congestión, pérdida de tiempo y emisiones adicionales. Este sistema entrega información de ocupación en tiempo real para reducir esa búsqueda (ver Avance 1 en `docs/`).

## Estructura del repositorio

```
├── README.md
├── FUENTES.md          Declaración de librerías, código externo y uso de IA
├── firmware/           Código del ESP32 (.ino)
├── hardware/           Esquemático, diagrama de conexiones y BOM
├── diseño-3d/          Encapsulado (Fusion 360 .f3d, STL), planos y renders
├── testing/            Protocolo de pruebas y datos de calibración
└── docs/               Informes y presentaciones
```

## Componentes necesarios

| Componente | Cantidad | Uso |
|---|---|---|
| ESP32 DevKit (USB-C) | 1 | Microcontrolador con WiFi |
| Sensor ultrasónico impermeable SR04M-2 (familia AJ-SR04M) | 1 | Medición de distancia |
| Módulo LED bicolor rojo/verde | 1 | Indicador de estado |
| Módulo buzzer pasivo | 1 | Aviso sonoro en cada cambio de estado |
| Resistencia 1 kΩ | 1 | En serie con la señal Echo |
| Resistencias 220 Ω | 2 | Limitación de corriente del LED |
| Breadboard 400 puntos y cables macho-hembra | 1 juego | Montaje del prototipo |
| Encapsulado impreso en 3D | 1 | Protección del circuito |

El detalle con especificaciones y costos está en `hardware/` (BOM).

## Conexiones

| Componente | Pin del componente | Conexión |
|---|---|---|
| Sensor SR04M-2 | VCC | 3,3 V |
| | GND | GND |
| | RX (Trig) | D5 |
| | TX (Echo) | Resistencia 1 kΩ en serie → D18 |
| LED bicolor | S | 220 Ω → D25 (verde) |
| | Pin del medio | 220 Ω → D26 (rojo) |
| | − | GND |
| Buzzer pasivo | S | D27 |
| | Pin del medio | 3,3 V |
| | − | GND |

## Cómo cargar el firmware

1. Instalar [Arduino IDE](https://www.arduino.cc/en/software).
2. Agregar el soporte para ESP32: en Archivo → Preferencias, pegar en "Gestor de URLs adicionales de tarjetas" la dirección `https://espressif.github.io/arduino-esp32/package_esp32_index.json`. Luego, en Herramientas → Placa → Gestor de tarjetas, instalar **esp32 de Espressif Systems**.
3. Abrir `firmware/sensor_estacionamiento.ino`.
4. Editar `WIFI_SSID` y `WIFI_PASS` con los datos de la red a usar (se recomienda el hotspot de un celular, ya que las redes universitarias suelen bloquear dispositivos IoT).
5. Seleccionar la placa **ESP32 Dev Module** y el puerto COM correspondiente.
6. Cargar el código. No se necesita instalar librerías adicionales: `WiFi.h` y `WebServer.h` vienen incluidas con el soporte de ESP32.
7. Abrir el Monitor Serie a 115200 baudios para ver la dirección IP y las lecturas del sensor.

## Cómo ver el dashboard

Al encender, el ESP32 hace una prueba de LED y buzzer y luego intenta conectarse a la red configurada:

- **Si se conecta:** el Monitor Serie muestra una dirección (por ejemplo `http://192.168.x.x`). Abrirla desde un celular conectado a la misma red.
- **Si no encuentra la red en 10 s:** el ESP32 crea su propia red WiFi llamada `Estacionamiento-A1`. Conectar el celular a esa red y abrir `http://192.168.4.1`.

La página muestra el estado (LIBRE en verde, OCUPADO en rojo), la distancia medida, el porcentaje de respuesta del sensor, el número de cambios de estado y el tiempo encendido. Se actualiza cada segundo. Los datos en formato JSON están disponibles en `/estado`.

## Funcionamiento

1. Cada ciclo toma 5 lecturas del sensor y usa la mediana para filtrar lecturas erráticas.
2. Se descartan lecturas menores a 20 cm (zona ciega del sensor).
3. La distancia se corrige con una calibración lineal obtenida por regresión (distancia real = 1,011 × medida + 4,0 cm; R² = 0,99997).
4. El estado cambia a OCUPADO bajo 60 cm y vuelve a LIBRE sobre 80 cm (histéresis, evita parpadeos en el límite).
5. Un cambio de estado solo se aplica tras 2 mediciones válidas consecutivas, para evitar falsos positivos (por ejemplo, un peatón que pasa).
6. LED verde indica libre, rojo indica ocupado; el buzzer emite un pitido en cada cambio.

Los umbrales están configurados para la demostración sobre una mesa y se ajustan en el código según la altura de instalación.
