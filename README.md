# Greywater Pump Controller

Control remoto, seguro y sin nube de la **bomba de aguas grises** de una vivienda, integrado a
[Home Assistant](https://www.home-assistant.io/). Hardware de bajo costo (ESP32 + módulo de relés)
y firmware declarativo en [ESPHome](https://esphome.io/).

![ESPHome](https://img.shields.io/badge/firmware-ESPHome-000000)
![ESP32](https://img.shields.io/badge/MCU-ESP32--WROOM--32U-blue)
![Home Assistant](https://img.shields.io/badge/integración-Home%20Assistant-41BDF5)
![License](https://img.shields.io/badge/license-MIT-green)

## Características

- **Encendido y apagado desde Home Assistant** (descubrimiento automático por mDNS, API cifrada).
- **Interfaz web local** en el propio equipo, para operar sin Home Assistant.
- **Fail-safe**: la carga va al contacto normalmente abierto y, tras un corte de luz, la bomba
  arranca **apagada** (`restore_mode: ALWAYS_OFF`). Nada se enciende solo.
- **Estado legible**: sensor de texto `ENCENDIDA - bombeando` / `APAGADA - no bombea`.
- **Actualización OTA cifrada** y punto de acceso de rescate si falla el WiFi.
- Telemetría de diagnóstico: señal WiFi, IP, botón de reinicio.
- 3 canales de relé adicionales disponibles para ampliar el sistema (válvulas, sensores, etc.).

## Arquitectura

```
 Home Assistant ──(API ESPHome cifrada, WiFi)──► ESP32-WROOM-32U ──GPIO32──► Relé 1 (NO) ──► Bomba
        ▲                                              │
        └──── interfaz web local (puerto 80) ◄─────────┘
```

## Hardware

| Componente | Detalle |
|---|---|
| Controlador | Devkit ESP32-WROOM-32U (antena externa U.FL) |
| Actuación | Módulo de 4 relés con optoacoplador (10 A / 250 V nominal) |
| Alimentación | 5 V al módulo de relés desde el pin `5V` del devkit |

### Asignación de pines

| Función | GPIO | Notas |
|---|---|---|
| Bomba (relé 1) | 32 | Salida segura: no es pin de arranque (*strapping*) ni emite pulsos al bootear |
| Canal 2 / 3 / 4 (libres) | 33 / 25 / 26 | Idem |
| LED de estado | 2 | Parpadea ante problemas de WiFi/API |
| Entradas analógicas libres | 34, 35, 36, 39 | ADC1 (los pines ADC2 no miden con WiFi activo) |

### Conexionado

`IN1..IN4` del módulo a los GPIO de la tabla, `GND` común y `VCC` a **5 V**. La bomba va al
**canal 1, contacto NO (normalmente abierto)**: si se pierde la alimentación del control, la bomba
queda parada.

> ⚠️ **Seguridad eléctrica.** El relé del kit declara 10 A, pero una bomba tiene un pico de
> arranque de varias veces su corriente nominal y las borneras del módulo son limitadas. Para
> bombas de más de ~½ HP, el relé debe comandar un **contactor**, no la bomba directamente.
> El cableado de 220 V debe hacerlo una persona competente, con protecciones (disyuntor
> diferencial y térmica) y en gabinete cerrado.

## Puesta en marcha

Requisitos: Python 3 y [ESPHome](https://esphome.io/guides/installing_esphome).

```bash
git clone https://github.com/bergamifer/greywater-pump-controller.git
cd greywater-pump-controller/esphome

cp secrets.example.yaml secrets.yaml      # completar WiFi y claves
openssl rand -base64 32                   # generar la clave de API

esphome run bomba.yaml --device /dev/ttyUSB0   # primera carga por USB
esphome run bomba.yaml                         # luego, OTA por WiFi
esphome logs bomba.yaml                        # logs por red
```

En Home Assistant el equipo aparece como **"Bomba aguas grises"** y pide la clave de API. Si HA
solicita permiso para controlar dispositivos ESPHome, hay que aceptarlo; de lo contrario las
entidades quedan *Unavailable*.

### Polaridad del relé

`rele_invertido: "true"` (activo-bajo) es lo habitual en módulos de 4 canales con optoacoplador.
Si el relé conmuta al revés, cambiar la constante en `esphome/bomba.yaml`. **Verificar en banco
antes de conectar la bomba.**

## Estados

| Home Assistant | Relé | Bomba |
|---|---|---|
| ENCENDIDA | bobina energizada, COM–NO cerrado | recibe tensión: bombea |
| APAGADA | contacto abierto | sin tensión: no bombea |

## Estructura del repositorio

```
├── esphome/
│   ├── bomba.yaml              # firmware
│   └── secrets.example.yaml    # plantilla de credenciales (secrets.yaml no se versiona)
├── docs/                       # esquemas y fotos de la instalación
└── hardware/                   # diagramas de conexionado
```

## Roadmap

- [ ] Apagado de seguridad por tiempo (*timeout*): que la bomba se apague sola tras N minutos si
      se pierde Home Assistant o el WiFi (evita trabajar en seco o vaciar de más).
- [ ] Sensor de nivel/humedad para operación automática.
- [ ] Reserva de IP en el router y dashboard dedicado en Home Assistant.
- [ ] Esquema de conexionado y fotos de la instalación.

## Licencia

[MIT](LICENSE)
