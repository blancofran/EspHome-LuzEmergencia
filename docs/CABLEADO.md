# Cableado del ESP8266 (NodeMCU)

Este documento detalla el cableado físico del proyecto: la placa usada, el
circuito divisor de voltaje que detecta la presencia de corriente, y la
conexión del relé. El circuito de detección parte de un **adaptador de 5V**
ya armado (cargador USB / fuente de pared) enchufado en la toma que se
quiere monitorear — no se cablea nada del lado de AC, el adaptador es un
dispositivo sellado que solo se enchufa.

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

## Antes de empezar

- El **adaptador de 5V** que monitorea la toma no requiere cableado de tu
  parte: es un dispositivo sellado, solo se enchufa.
- Si el relé va a conmutar directamente la alimentación de la luz de
  emergencia (cortando/uniendo sus propios cables de línea en vez de usar un
  enchufe intermedio), esa parte sí es de línea AC: desenergiza el circuito
  en el breaker, verifica con multímetro que no haya tensión, y si no tienes
  experiencia con instalaciones eléctricas contrata a un electricista para
  esa conexión puntual.
- Todo lo demás (NodeMCU, divisor, lógica del relé) es de bajo voltaje y
  seguro de armar tú mismo.
- Usa un gabinete cerrado, no conductor, para todo el conjunto una vez
  cableado.

## Diagrama general

```
   Adaptador 5V ──────────────────────┐
   (enchufado en la toma               │ +5V         GND
    que se quiere monitorear)          │              │
                                       │              │
                            ┌──────────┴──┐           │
                            │             │           │
                       R1 10kΩ            │           │
                            │             │           │
                            ├──── A0 (NodeMCU)         │
                            │             │           │
                       R2 15kΩ       C1 0.1µF          │
                            │             │           │
                            └──────┬──────┘           │
                                   │                   │
                                  GND ──────────────────┴──── GND (común)


   Fuente de respaldo (batería / UPS) ──── 5V·VIN, GND ──── NodeMCU + Relé


   D2 (GPIO4) NodeMCU ──── IN del relé ──── COM/NO del relé ──── Luz de emergencia
```

> El adaptador de 5V solo alimenta el divisor de voltaje (el "sensor"). La
> NodeMCU y el relé se alimentan de una fuente de respaldo aparte — ver la
> sección más abajo. El relé y la NodeMCU comparten GND, y la NodeMCU lo
> controla con la señal digital `D2` (`GPIO4`).

## Divisor de voltaje (detección de corriente)

Componentes: **R1 = 10 kΩ**, **R2 = 15 kΩ**, **C1 = 0.1 µF**, alimentados
desde la salida de 5V del **adaptador** enchufado en la toma monitoreada.

```
  +5V (adaptador) ──[ R1 10kΩ ]──┬── A0 (NodeMCU)
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

Voltaje resultante en el pin `A0` con el adaptador energizado:

```
V(A0) = 5V × R2 / (R1 + R2) = 5V × 15k / (10k + 15k) = 3.0V
```

3.0V queda por debajo del máximo admitido por el pin `A0` de la NodeMCU
(~3.3V), dejando margen de seguridad. Cuando falta la corriente en la toma
monitoreada, el adaptador deja de entregar 5V y `V(A0)` cae a ~0V, que es lo
que el YAML usa como umbral (`multiply: 5` + comparación en `0.5`) para
decidir si hay o no corriente.

**Verifica siempre con un multímetro** el voltaje real en `A0` antes de
conectarlo a la NodeMCU, y ajusta R1/R2 si tu adaptador o las tolerancias de
las resistencias dan un valor distinto — nunca debe superar 3.3V.

## Relé

- Usa un módulo de relé para 5V con **opto-acoplador** (aísla eléctricamente
  la lógica de la NodeMCU del lado de conmutación).
- `IN` del relé → `D2` (`GPIO4`) de la NodeMCU.
- `VCC` del relé → 5V de la fuente de respaldo (junto con la NodeMCU).
- `GND` del relé → `GND` común con la NodeMCU.
- Revisa si tu módulo de relé es activo en bajo (la mayoría de los módulos
  de 1 canal lo son). Si el relé enciende la luz al revés de lo esperado
  (encendida cuando debería estar apagada), cambia `inverted: true` en el
  `switch` `relay0` del YAML.
- El contacto `COM`/`NO` del relé conmuta la alimentación de la luz de
  emergencia. Si la conectas mediante un enchufe/toma intermedia no hay
  cables de línea que manipular; si en cambio empalmas directamente los
  cables de la luz, tratá esa conexión como trabajo de línea AC (ver
  "Antes de empezar").

## Alimentación de respaldo — requisito de diseño

La NodeMCU y el relé necesitan estar **energizados durante el corte** para
poder encender la luz en el momento en que ocurre. Si los alimentas
únicamente desde el mismo adaptador de 5V que estás monitoreando, el corte
de corriente también los apaga justo cuando deben actuar, y el sistema no
podrá encender el relé.

Alimenta la NodeMCU y el relé desde una fuente independiente que se
mantenga activa durante un corte: una batería/power bank USB, un pequeño
UPS, o la misma batería de respaldo de la luz de emergencia (si la tiene y
puede entregar 5V/500mA). El adaptador de 5V monitoreado solo debería usarse
para alimentar el lado de **medición** (el divisor de voltaje), no la
NodeMCU completa.

## Checklist de armado

1. Arma el divisor R1/R2/C1 en una zona de baja tensión.
2. Enchufa el adaptador de 5V en la toma que quieres monitorear y, antes de
   conectar nada a la NodeMCU, mide con multímetro el voltaje en el nodo del
   divisor — confirma que sea ~3V y nunca mayor a 3.3V.
3. Conecta `A0` y `GND` del divisor a la NodeMCU.
4. Cablea el relé: `IN` a `D2`, `VCC`/`GND` a la fuente de respaldo elegida
   junto con la NodeMCU.
5. Conecta `COM`/`NO` del relé a la alimentación de la luz de emergencia
   (por enchufe intermedio, o empalmando línea si corresponde — ver
   "Antes de empezar" para las precauciones de esa conexión puntual).
6. Cierra el gabinete manteniendo el divisor y el relé ordenados y
   protegidos de contacto accidental.
7. Energiza todo y verifica en Home Assistant que "Electricidad" refleje el
   estado real, y que al provocar un corte (desenchufando el adaptador de
   5V monitoreado) el relé encienda tras el retardo configurado.
