# Luz Emergencia

Proyecto [ESPHome](https://esphome.io/) que detecta cortes de electricidad y
enciende automáticamente una luz de emergencia (relé), con integración
completa en Home Assistant. Soporta **ESP8266 (NodeMCU)** o **ESP32** — el
diseño eléctrico es el mismo para ambos, solo cambia el archivo YAML y
algunos pines.

## Qué hace

El dispositivo mide continuamente la tensión de la red eléctrica a través de
un circuito divisor de voltaje conectado al ADC del microcontrolador. Cuando detecta
que la tensión cae por debajo del umbral (corte de electricidad), enciende un
relé que activa una luz de emergencia. Cuando la corriente regresa, el relé
se apaga automáticamente tras un retardo configurable, para evitar apagones
por parpadeos momentáneos de la red.

El encendido de la luz también puede ajustarse según la hora del día (solo de
noche, o también de día si el usuario lo permite) y puede forzarse al llegar
el atardecer si ya había un corte en curso.

## Hardware requerido

Elegí **una** de estas dos placas — el resto del hardware es idéntico para
ambas:

- **ESP8266** en una placa **NodeMCU** (módulo ESP-12E, board `nodemcuv2` en
  ESPHome) — no un módulo ESP-01, que no expone los pines `A0`/`GPIO4` que
  usa este proyecto. YAML: [luz-emergencia.yaml](luz-emergencia.yaml).
  Cableado: [docs/CABLEADO-ESP8266.md](docs/CABLEADO-ESP8266.md).
- **ESP32 DevKit** (board `esp32dev` en ESPHome). YAML:
  [luz-emergencia-esp32.yaml](luz-emergencia-esp32.yaml). Cableado:
  [docs/CABLEADO-ESP32.md](docs/CABLEADO-ESP32.md).

Y en ambos casos:

- **Relé** para controlar la luz de emergencia (conectado a `GPIO4` — pin
  `D2` en la serigrafía de la NodeMCU; en ESP32 es el mismo número de pin
  pero en otra ubicación física, ver [docs/CABLEADO-ESP32.md](docs/CABLEADO-ESP32.md)).
- **Luz de emergencia de 12V DC**, conmutada por el contacto del relé.
- **Batería de respaldo de 12V**, independiente del adaptador monitoreado,
  que alimenta toda la electrónica (microcontrolador, relé) y la luz de
  emergencia — así el sistema sigue funcionando durante el corte que debe
  detectar.
- **Controlador de carga** (AC → 12V), enchufado en la misma toma
  monitoreada, que mantiene cargada la batería de respaldo.
- **Regulador step-down 12V → 5V** (tipo LM2596 o similar) entre la batería
  y la lógica del microcontrolador/relé.
- **Circuito divisor de voltaje** para medir la presencia de tensión de línea
  de forma segura, partiendo de un adaptador ya armado:
  - **Adaptador de 5V** (cargador USB / fuente de pared) enchufado en la
    toma que se quiere monitorear, para obtener una señal de bajo voltaje
    proporcional a la presencia de corriente. Al ser un dispositivo sellado,
    no requiere cablear nada del lado de AC.
  - **R1 = 10 kΩ** y **R2 = 15 kΩ** formando el divisor de voltaje hacia el
    pin ADC (`A0` en NodeMCU, `GPIO34` en ESP32 — en ESP32 además hay que
    declarar `attenuation: 11db` en el sensor, ver
    [docs/CABLEADO-ESP32.md](docs/CABLEADO-ESP32.md)).
  - **C1 = 0.1 µF** como filtro/estabilizador de la lectura hacia el pin ADC.

Para el diagrama de cableado completo (incluyendo el regulador y la conexión
de la batería al relé), el cálculo del divisor y el checklist de armado paso
a paso, ver **[docs/CABLEADO-ESP8266.md](docs/CABLEADO-ESP8266.md)** (NodeMCU/ESP8266) o
**[docs/CABLEADO-ESP32.md](docs/CABLEADO-ESP32.md)** (ESP32).

### ⚠️ Advertencia de seguridad

Todo el proyecto es de bajo voltaje DC (5V/12V) — no hay cables de línea AC
que manipular en ningún punto. Los dos únicos componentes que tocan AC (el
adaptador de 5V que monitorea la toma, y el controlador de carga de la
batería) son dispositivos sellados que solo se enchufan.

Aun así, la batería de 12V puede entregar corriente suficiente para
provocar chispas o daño si se cortocircuitan sus terminales:

- Verifica siempre la **polaridad** antes de conectar cada componente
  (regulador, relé, luz).
- No dejes los terminales de la batería expuestos o al alcance de objetos
  metálicos.
- Usa un gabinete cerrado, no conductor, para todo el conjunto una vez
  armado.

## Pines usados

**NodeMCU / ESP8266** ([luz-emergencia.yaml](luz-emergencia.yaml)):

| Pin (YAML) | Etiqueta en la NodeMCU | Función                                              |
|------------|-------------------------|-------------------------------------------------------|
| `A0`       | `A0`                    | Entrada ADC — lectura de voltaje del divisor          |
| `GPIO4`    | `D2`                    | Salida digital — control del relé de luz de emergencia |

**ESP32** ([luz-emergencia-esp32.yaml](luz-emergencia-esp32.yaml)):

| Pin (YAML) | Función                                                        |
|------------|-----------------------------------------------------------------|
| `GPIO34`   | Entrada ADC1 — lectura de voltaje del divisor (con `attenuation: 11db`) |
| `GPIO4`    | Salida digital — control del relé de luz de emergencia          |

Detalle completo de cableado en [docs/CABLEADO-ESP8266.md](docs/CABLEADO-ESP8266.md) o
[docs/CABLEADO-ESP32.md](docs/CABLEADO-ESP32.md).

## Instalación

1. Copia el archivo de ejemplo de secretos:

   ```bash
   cp secrets.yaml.example secrets.yaml
   ```

2. Edita `secrets.yaml` y completa tus propios valores (SSID y password de
   WiFi, password del punto de acceso de respaldo, latitud/longitud de tu
   ubicación).

3. Genera una clave de encriptación única para la API de Home Assistant:

   ```bash
   openssl rand -base64 32
   ```

   Copia el resultado en el campo `api_encryption_key` de `secrets.yaml`.

4. Compila y flashea el dispositivo con ESPHome, usando el YAML que
   corresponda a tu placa:

   ```bash
   # NodeMCU / ESP8266
   esphome run luz-emergencia.yaml

   # ESP32
   esphome run luz-emergencia-esp32.yaml
   ```

   `secrets.yaml` es el mismo para ambos — solo cambia el archivo principal.

## Entidades configurables desde Home Assistant

Una vez instalado, el dispositivo expone las siguientes entidades ajustables
desde Home Assistant:

- **Retardo de apagado tras regreso de corriente** — tiempo (segundos) que
  debe mantenerse la corriente antes de apagar el relé.
- **Retardo para encender** — tiempo (milisegundos) que debe faltar la
  corriente antes de considerar que hay un corte real (evita falsos
  positivos por caídas momentáneas).
- **Permitir encendido diurno** — si está activo, la luz también se enciende
  durante el día en caso de corte (por defecto solo enciende de noche).
- **Encender al atardecer si hay corte** — si ya hay un corte en curso al
  llegar el atardecer, fuerza el encendido de la luz aunque el encendido
  diurno esté desactivado.

## Lógica de funcionamiento

- El relé se **enciende** cuando se detecta un corte de electricidad **y**
  además se cumple alguna de estas condiciones: es de noche, el usuario
  habilitó el encendido diurno, o acaba de llegar el atardecer con el corte
  ya en curso.
- El relé se **apaga** cuando la corriente regresa, después de esperar el
  tiempo configurado en "Retardo de apagado tras regreso de corriente" (para
  evitar apagados prematuros ante fluctuaciones breves de la red).

## Licencia

MIT. Ver el texto completo en [LICENSE](LICENSE) o en
https://opensource.org/licenses/MIT.
