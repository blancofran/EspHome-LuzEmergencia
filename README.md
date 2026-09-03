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

- **ESP8266** en una placa **NodeMCU** (módulo ESP-12E, board `nodemcuv2` en
  ESPHome) — no un módulo ESP-01, que no expone los pines `A0`/`GPIO4` que
  usa este proyecto.
- **Relé** para controlar la luz de emergencia (conectado a `GPIO4`, pin
  `D2` en la serigrafía de la NodeMCU).
- **Circuito divisor de voltaje** para medir la presencia de tensión de línea
  de forma segura, partiendo de un adaptador ya armado:
  - **Adaptador de 5V** (cargador USB / fuente de pared) enchufado en la
    toma que se quiere monitorear, para obtener una señal de bajo voltaje
    proporcional a la presencia de corriente. Al ser un dispositivo sellado,
    no requiere cablear nada del lado de AC.
  - **R1 = 10 kΩ** y **R2 = 15 kΩ** formando el divisor de voltaje hacia el
    pin ADC.
  - **C1 = 0.1 µF** como filtro/estabilizador de la lectura hacia `A0`.
  - Una **fuente de respaldo** (batería/power bank/UPS) separada, para
    alimentar la NodeMCU y el relé de forma que sigan activos durante un
    corte — ver detalle en [docs/CABLEADO.md](docs/CABLEADO.md).

Para el diagrama de cableado completo, el cálculo del divisor y el checklist
de armado paso a paso, ver **[docs/CABLEADO.md](docs/CABLEADO.md)**.

### ⚠️ Advertencia de seguridad

El adaptador de 5V que monitorea la toma no requiere cableado — es un
dispositivo sellado, solo se enchufa. El único punto que puede involucrar
**voltaje de línea AC (110-120V)** es la salida del relé, si conectas la luz
de emergencia empalmando directamente sus cables en vez de usar un enchufe
intermedio:

- **Desenergiza el circuito** (apaga el interruptor/breaker correspondiente)
  antes de hacer esa conexión.
- Verifica con un multímetro que no haya tensión antes de tocar los
  conductores.
- Si no tienes experiencia trabajando con instalaciones eléctricas de línea,
  **contrata a un electricista calificado** para esa conexión puntual. El
  resto del proyecto (NodeMCU, relé, divisor) es de bajo voltaje y seguro de
  armar tú mismo.

## Pines usados

| Pin (YAML) | Etiqueta en la NodeMCU | Función                                              |
|------------|-------------------------|-------------------------------------------------------|
| `A0`       | `A0`                    | Entrada ADC — lectura de voltaje del divisor          |
| `GPIO4`    | `D2`                    | Salida digital — control del relé de luz de emergencia |

Detalle completo de cableado en [docs/CABLEADO.md](docs/CABLEADO.md).

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
