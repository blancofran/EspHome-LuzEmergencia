# Cableado del ESP32 (DevKit)

Este documento es la variante para **ESP32** de [CABLEADO.md](CABLEADO.md)
(la versión para NodeMCU/ESP8266). El diseño eléctrico es exactamente el
mismo — mismo divisor, misma batería, mismo cargador, mismo relé — lo único
que cambia son los pines y un parámetro del sensor ADC en el YAML
([luz-emergencia-esp32.yaml](../luz-emergencia-esp32.yaml)). Si dudás entre
placas, ver la comparación en el [README](../README.md#hardware-requerido).

## Placa usada

Un **ESP32 DevKit** genérico (el módulo ancho de 30/38 pines con USB micro
y dos botones EN/BOOT). En ESPHome corresponde al board `esp32dev`.

A diferencia de la NodeMCU, el ESP32 **no tiene un divisor incorporado** en
sus pines ADC — el pin que uses queda expuesto directo al chip, con el
mismo máximo de ~3.3V. Esto no cambia el circuito (el divisor R1/R2/C1 ya
apunta a ~3V, con margen), pero sí importa elegir un pin ADC1 (los pines
ADC2 comparten hardware con el WiFi y pueden dar lecturas erráticas con
WiFi activo).

| Pin usado en el YAML | Función                                                    |
|-----------------------|-------------------------------------------------------------|
| `GPIO34`               | Entrada ADC1, solo lectura — lectura del divisor            |
| `GPIO4`                | Salida digital — control del relé (no es pin de "strapping") |
| `5V` / `VIN`           | Alimentación de la propia placa (según el modelo)            |
| `GND`                  | Referencia común                                             |

`GPIO34` es una entrada **solo-lectura** (no sirve como salida), lo cual es
ideal acá porque el divisor solo necesita ser leído. `GPIO4` es una salida
de propósito general válida en el ESP32 — no es uno de los pines de
"strapping" (`GPIO0`, `GPIO2`, `GPIO5`, `GPIO12`, `GPIO15`) que definen el
modo de arranque y conviene evitar para salidas. El número de pin es el
mismo que en la versión NodeMCU, pero su ubicación física en la placa es
distinta — revisá la serigrafía de tu ESP32 DevKit específico, ya que varía
entre fabricantes (a diferencia de la NodeMCU, el ESP32 DevKit no tiene un
pinout tan estandarizado).

## Antes de empezar

- El **adaptador de 5V** que monitorea la toma y el **controlador de
  carga** de la batería no requieren cableado de tu parte: son dispositivos
  sellados, solo se enchufan.
- La luz de emergencia funciona a **12V DC** desde la batería de respaldo, a
  través del relé — no hay cables de línea AC en ningún punto del proyecto.
- Aun así, trabajá con cuidado: una batería de 12V puede entregar corriente
  suficiente para generar chispas o daño si se cortocircuitan sus bornes.
  Verificá la polaridad antes de conectar cada componente.
- Usa un gabinete cerrado, no conductor, para todo el conjunto una vez
  cableado.

## Diagrama general

![Diagrama de cableado: adaptador de 5V al divisor R1/R2/C1 hacia GPIO34 (ADC1) de un ESP32 DevKit; un controlador de carga (AC → 12V) mantiene cargada una batería de 12V independiente, que alimenta, a través de un regulador step-down, al ESP32 y la lógica del relé, y directamente al contacto COM del relé, que conmuta 12V hacia la luz de emergencia; GND común entre todos los bloques.](cableado-esp32.svg)

*El adaptador de 5V solo alimenta el divisor de voltaje (el "sensor"). El
controlador de carga toma AC de la misma toma monitoreada y mantiene
cargada la batería de 12V — de ahí sale toda la energía del resto del
sistema: el ESP32 y la lógica del relé a través de un regulador step-down
a 5V, y la luz directamente en 12V a través del contacto del relé. El
relé y el ESP32 comparten GND, y el ESP32 controla el relé con la señal
digital `GPIO4`.*

## Divisor de voltaje (detección de corriente)

Componentes: **R1 = 10 kΩ**, **R2 = 15 kΩ**, **C1 = 0.1 µF**, alimentados
desde la salida de 5V del **adaptador** enchufado en la toma monitoreada
(ver el detalle de R1/R2/C1 en el diagrama de arriba) — idéntico al
circuito de la versión NodeMCU.

```
V(GPIO34) = 5V × R2 / (R1 + R2) = 5V × 15k / (10k + 15k) = 3.0V
```

3.0V queda por debajo del máximo admitido por los pines ADC del ESP32
(~3.3V), dejando margen de seguridad. Cuando falta la corriente en la toma
monitoreada, el adaptador deja de entregar 5V y `V(GPIO34)` cae a ~0V.

**Diferencia clave con la versión NodeMCU:** por defecto el ADC del ESP32
en ESPHome solo lee bien hasta ~1.1V — para que los 3.0V del divisor entren
en rango hay que declarar la atenuación en el sensor:

```yaml
sensor:
  - platform: adc
    pin: GPIO34
    attenuation: 11db   # habilita el rango completo, 0–3.3V
    id: volt
    ...
```

(esto ya está incluido en [luz-emergencia-esp32.yaml](../luz-emergencia-esp32.yaml)).

**Verifica siempre con un multímetro** el voltaje real en `GPIO34` antes de
conectarlo a la placa, y ajusta R1/R2 si tu adaptador o las tolerancias de
las resistencias dan un valor distinto — nunca debe superar 3.3V.

## Relé

- Usa un módulo de relé para 5V con **opto-acoplador** (aísla eléctricamente
  la lógica del ESP32 del lado de conmutación).
- `IN` del relé → `GPIO4` del ESP32.
- `VCC` del relé → salida de 5V del **regulador step-down** (ver más abajo).
- `GND` del relé → `GND` común con el ESP32 y la batería.
- `COM` del relé → **+12V directo de la batería** (sin pasar por el
  regulador — el contacto conmuta la tensión completa de la batería).
- `NO` del relé → terminal **+** de la luz de emergencia. El terminal **–**
  de la luz vuelve al `GND` común (a la batería).
- Revisa si tu módulo de relé es activo en bajo (la mayoría de los módulos
  de 1 canal lo son). Si el relé enciende la luz al revés de lo esperado,
  cambia `inverted: true` en el `switch` `relay0` del YAML.

## Regulador 5V (step-down 12V → 5V)

El ESP32 y la lógica del relé trabajan a 5V, pero la batería de respaldo es
de 12V. Un regulador step-down (buck) tipo LM2596 o similar convierte ese
12V a un 5V estable — idéntico al de la versión NodeMCU:

- Entrada: `+12V` y `GND` de la batería de respaldo.
- Salida: `5V` y `GND` hacia el pin `5V`/`VIN`/`GND` del ESP32 y `VCC`/`GND`
  del relé.
- Elegí un módulo que entregue al menos 500 mA.
- El contacto `COM`/`NO` del relé **no** pasa por este regulador: se
  alimenta directo de los 12V de la batería.

## Controlador de carga (AC → 12V)

Igual que en la versión NodeMCU: un módulo sellado que se enchufa directo a
AC en la **misma toma que estás monitoreando** con el adaptador de 5V, y
mantiene cargada la batería de 12V. Ver el detalle completo en la sección
"Controlador de carga" de [CABLEADO.md](CABLEADO.md) — no cambia nada de
esta parte al usar ESP32.

## Alimentación de respaldo — requisito de diseño

El ESP32 y el relé necesitan estar **energizados durante el corte** para
poder encender la luz en el momento en que ocurre. Por eso todo el sistema
—ESP32, relé y la luz de emergencia misma— se alimenta de una **batería de
respaldo de 12V**, independiente del adaptador monitoreado. El adaptador de
5V solo alimenta el lado de **medición** (el divisor de voltaje), nunca el
ESP32 ni la luz.

## Checklist de armado

1. Arma el divisor R1/R2/C1 en una zona de baja tensión.
2. Enchufa el adaptador de 5V en la toma que quieres monitorear y, antes de
   conectar nada al ESP32, mide con multímetro el voltaje en el nodo del
   divisor — confirma que sea ~3V y nunca mayor a 3.3V.
3. Conecta `GPIO34` y `GND` del divisor al ESP32. Verificá que el YAML
   tenga `attenuation: 11db` en el sensor ADC.
4. Conecta el controlador de carga a la batería (`+12V`/`GND`) y enchufalo
   en la misma toma monitoreada. Con la batería conectada, medí con
   multímetro que esté cargando.
5. Cablea el regulador step-down: entrada `+12V`/`GND` a la batería, salida
   `5V`/`GND` al pin `5V`/`VIN` y `GND` del ESP32, y `VCC`/`GND` del relé.
   Antes de conectar el ESP32, medí con multímetro que la salida sea ~5V.
6. Cablea el relé: `IN` a `GPIO4`, `COM` directo a `+12V` de la batería,
   `NO` a la entrada `+` de la luz de emergencia, y la `–` de la luz de
   vuelta al `GND` común.
7. Cierra el gabinete manteniendo el divisor, el controlador de carga, el
   regulador y el relé ordenados y protegidos de contacto accidental.
8. Compila y flasheá con `luz-emergencia-esp32.yaml` (no el de NodeMCU).
   Energiza todo y verifica en Home Assistant que "Electricidad" refleje el
   estado real, y que al provocar un corte (desenchufando el adaptador de
   5V monitoreado) el relé encienda tras el retardo configurado.
