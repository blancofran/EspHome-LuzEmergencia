# Cableado del ESP8266 (NodeMCU)

Este documento detalla el cableado físico completo del proyecto: la placa
usada, el circuito divisor de voltaje que detecta la presencia de corriente,
y la conexión del relé. Léelo junto con la [advertencia de seguridad del
README](../README.md#advertencia-de-seguridad) antes de empezar.

## Placa usada

El proyecto está pensado para una placa de desarrollo **NodeMCU** basada en
el módulo **ESP-12E** (la típica placa ancha de 30 pines con USB micro,
botones RST/FLASH y pines serigrafiados `D0`–`D8` y `A0`), no para el módulo
ESP-01 de 8 pines. En ESPHome corresponde al board `nodemcuv2`.

Esto importa porque el ESP-01 **no expone** el pin ADC (`A0`) ni `GPIO4` en
su header — con ese módulo este proyecto no se podría cablear tal cual está.
La NodeMCU sí expone ambos pines, y además su pin `A0` ya incluye en la
propia placa un divisor de voltaje que permite entregarle hasta ~3.3V de
forma segura (el chip ESP8266 desnudo solo admite ~1V en su ADC interno).
Esto es clave para el cálculo del divisor externo más abajo.

| Pin usado en el YAML | Etiqueta en la placa NodeMCU | Función                          |
|-----------------------|-------------------------------|-----------------------------------|
| `A0`                  | `A0`                          | Entrada ADC — lectura del divisor |
| `GPIO4`                | `D2`                          | Salida digital — control del relé |
| `3V3` / `VIN`          | `3V3` / `Vin`                 | Alimentación de la propia NodeMCU |
| `GND`                  | `GND`                         | Referencia común                  |

## ⚠️ Antes de tocar cualquier cable

- **Desenergiza el circuito** en el interruptor/breaker antes de conectar o
  desconectar nada del lado de línea AC (110-120V).
- Verifica con un multímetro que no haya tensión antes de tocar conductores.
- Separa físicamente la sección de alta tensión (AC, HLK-PM01) de la sección
  de bajo voltaje (NodeMCU, divisor, lógica del relé) dentro del gabinete —
  usa canaletas o barreras aislantes entre ambas.
- Si no tienes experiencia con instalaciones de línea AC, **contrata un
  electricista calificado** para la parte de conexión a la red y la fuente
  aislada. El resto (NodeMCU, relé, divisor) es de bajo voltaje y seguro de
  armar tú mismo.
- Usa un gabinete cerrado, no conductor, para todo el conjunto una vez
  cableado.

## Diagrama general

```
  ── Sección AC (110-120V) ── ⚠ PELIGRO ⚠ ─────────────────────────────────
                                                             │
   Línea AC ──┬─────────────────────────────┐               │
   (L, N)     │                             │               │
              ▼                             ▼               │
        ┌───────────┐                 ┌───────────┐         │
        │ HLK-PM01  │                 │   RELÉ    │         │
        │  AC → 5V  │                 │  COM ─────┼─── L ───┘
        │  aislado  │                 │  NO  ──────── Luz de emergencia
        └─────┬─────┘                 └─────┬─────┘
              │ +5V   GND                    │  Vcc / GND / IN
              │                              │
  ── Sección de bajo voltaje ───────────────────────────────────────────────
              │                              │
              ├──────────────┐               │
              │              │               │
         R1 10kΩ             │        NodeMCU: 5V/VIN, GND
              │              │               │
              ├─── A0 (NodeMCU)              │
              │              │               │
         R2 15kΩ         C1 0.1µF            │
              │              │               │
              └──────┬───────┘               │
                     │                       │
                    GND ─────────────────────┴──── GND (común)

                                    D2 (GPIO4) NodeMCU ──── IN del relé
```

> El relé y la NodeMCU comparten GND. La NodeMCU controla el relé con la
> señal digital `D2` (`GPIO4`); el relé, a su vez, conmuta la luz de
> emergencia del lado de AC, físicamente aislado del lado de control.

## Divisor de voltaje (detección de corriente)

Componentes: **R1 = 10 kΩ**, **R2 = 15 kΩ**, **C1 = 0.1 µF**, alimentados
desde la salida de 5V del **HLK-PM01**.

```
  +5V (HLK-PM01) ──[ R1 10kΩ ]──┬── A0 (NodeMCU)
                                │
                          [ R2 15kΩ ]
                                │
                          [ C1 0.1µF ]
                                │
                               GND
```

- R1 conecta el nodo de medición con el riel de +5V.
- R2 conecta el mismo nodo a GND, formando el divisor.
- C1 filtra ruido/parpadeos y estabiliza la lectura del ADC.

Voltaje resultante en el pin `A0` con el HLK-PM01 energizado:

```
V(A0) = 5V × R2 / (R1 + R2) = 5V × 15k / (10k + 15k) = 3.0V
```

3.0V queda por debajo del máximo admitido por el pin `A0` de la NodeMCU
(~3.3V), dejando margen de seguridad. Cuando falta la corriente, el
HLK-PM01 deja de entregar 5V y `V(A0)` cae a ~0V, que es lo que el YAML usa
como umbral (`multiply: 5` + comparación en `0.5`) para decidir si hay o no
corriente.

**Verifica siempre con un multímetro** el voltaje real en `A0` antes de
conectarlo a la NodeMCU, y ajusta R1/R2 si tu HLK-PM01 o tolerancias de
resistencias dan un valor distinto — nunca debe superar 3.3V.

## Relé

- Usa un módulo de relé para 5V con **opto-acoplador** (aísla eléctricamente
  la lógica de la NodeMCU del lado de conmutación AC).
- `IN` del relé → `D2` (`GPIO4`) de la NodeMCU.
- `VCC` del relé → `VIN` (5V) de la NodeMCU (o directamente al +5V del
  HLK-PM01, según el módulo).
- `GND` del relé → `GND` común con la NodeMCU.
- Revisa si tu módulo de relé es activo en bajo (la mayoría de los módulos
  de 1 canal lo son). Si el relé enciende la luz al revés de lo esperado
  (encendida cuando debería estar apagada), cambia `inverted: true` en el
  `switch` `relay0` del YAML.
- El contacto `COM`/`NO` del relé conmuta la **fase (L)** hacia la luz de
  emergencia — nunca el neutro.

## ⚠️ Alimentación de respaldo — requisito de diseño

La NodeMCU y el relé necesitan estar **energizados durante el corte** para
poder encender la luz en el momento en que ocurre. Si alimentas la NodeMCU
únicamente desde el mismo HLK-PM01 que estás monitoreando, el corte de
corriente también apaga la NodeMCU justo cuando debe actuar, y el sistema no
podrá encender el relé.

Alimenta la NodeMCU y el relé desde una fuente independiente que se
mantenga activa durante un corte: una batería/power bank USB, un pequeño
UPS, o la misma batería de respaldo de la luz de emergencia (si la tiene y
puede entregar 5V/500mA). El HLK-PM01 solo debería usarse para alimentar el
lado de **medición** (el divisor de voltaje), no la NodeMCU completa.

## Checklist de armado

1. Cablea el HLK-PM01 (lado AC) con el circuito desenergizado. Verifica con
   multímetro que no haya tensión antes de continuar.
2. Arma el divisor R1/R2/C1 en una zona de baja tensión, alimentado desde el
   +5V del HLK-PM01.
3. Antes de conectar el divisor al `A0` de la NodeMCU, energiza solo el
   HLK-PM01 y mide con multímetro el voltaje en el nodo del divisor —
   confirma que sea ~3V y nunca mayor a 3.3V.
4. Conecta `A0` y `GND` del divisor a la NodeMCU.
5. Cablea el relé: `IN` a `D2`, `VCC`/`GND` a la fuente de respaldo elegida
   junto con la NodeMCU.
6. Cablea `COM`/`NO` del relé a la fase de la luz de emergencia, con el
   circuito de AC desenergizado.
7. Cierra el gabinete manteniendo separadas las secciones de AC y de bajo
   voltaje.
8. Energiza todo y verifica en Home Assistant que "Electricidad" refleje el
   estado real, y que al provocar un corte (apagando el breaker que
   alimenta el HLK-PM01) el relé encienda tras el retardo configurado.
