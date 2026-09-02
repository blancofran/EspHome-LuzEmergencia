# Luz Emergencia

Proyecto [ESPHome](https://esphome.io/) para un ESP8266 que detecta cortes de
electricidad y enciende automáticamente una luz de emergencia (relé), con
integración completa en Home Assistant.

## Qué hace

El dispositivo mide continuamente la tensión de la red eléctrica a través de
un circuito divisor de voltaje conectado al ADC del ESP8266. Cuando detecta
que la tensión cae por debajo del umbral (corte de electricidad), enciende un
relé que activa una luz de emergencia. Cuando la corriente regresa, el relé
se apaga automáticamente tras un retardo configurable, para evitar apagones
por parpadeos momentáneos de la red.

El encendido de la luz también puede ajustarse según la hora del día (solo de
noche, o también de día si el usuario lo permite) y puede forzarse al llegar
el atardecer si ya había un corte en curso.

## Hardware requerido

- **ESP8266** (board `esp01_1m`, p. ej. ESP-01).
- **Relé** para controlar la luz de emergencia (conectado a `GPIO4`).
- **Circuito divisor de voltaje** para medir la presencia de tensión de línea
  de forma segura y aislada:
  - Fuente aislada tipo **HLK-PM01** (5V) alimentada desde la línea AC,
    para obtener una señal de bajo voltaje proporcional a la presencia de
    corriente.
  - **R1 = 10 kΩ** y **R2 = 15 kΩ** formando el divisor de voltaje hacia el
    pin ADC.
  - **C1 = 0.1 µF** como filtro/estabilizador de la lectura hacia `A0`.

### ⚠️ Advertencia de seguridad

Este proyecto involucra cableado cerca de **voltaje de línea AC (110-120V)**,
que puede causar lesiones graves o la muerte si se manipula incorrectamente.

- **Desenergiza el circuito** (apaga el interruptor/breaker correspondiente)
  antes de hacer cualquier conexión eléctrica.
- Verifica con un multímetro que no haya tensión antes de tocar los
  conductores.
- Si no tienes experiencia trabajando con instalaciones eléctricas de línea,
  **contrata a un electricista calificado** para la parte de conexión a la
  red AC y la fuente aislada. La parte de bajo voltaje (ESP8266, relé, lógica
  de ESPHome) puedes montarla tú mismo con seguridad.
- Usa siempre una fuente **aislada galvánicamente** (como el HLK-PM01) entre
  la línea AC y el circuito de control; nunca conectes el ADC del ESP8266
  directamente a la red eléctrica.

## Pines usados

| Pin   | Función                                              |
|-------|-------------------------------------------------------|
| `A0`  | Entrada ADC — lectura de voltaje del divisor          |
| `GPIO4` | Salida digital — control del relé de luz de emergencia |

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

4. Compila y flashea el dispositivo con ESPHome:

   ```bash
   esphome run luz-emergencia.yaml
   ```

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
