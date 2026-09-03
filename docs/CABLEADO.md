# Cableado del ESP8266 (NodeMCU)

Este documento detalla el cableado físico del proyecto: la placa usada, el
circuito divisor de voltaje que detecta la presencia de corriente, y la
conexión del relé. El circuito de detección parte de un **adaptador de 5V**
ya armado (cargador USB / fuente de pared) enchufado en la toma que se
quiere monitorear — no se cablea nada del lado de AC, el adaptador es un
dispositivo sellado que solo se enchufa. Todo lo demás (NodeMCU, relé y la
luz de emergencia) se alimenta de una **batería de respaldo de 12V**,
mantenida cargada por un **controlador de carga** (también un dispositivo
sellado, enchufado en la misma toma monitoreada). En todo el proyecto no hay
ningún cable de línea AC que manipular — los dos únicos puntos que tocan AC
(el adaptador de 5V y el controlador de carga) son dispositivos ya armados
que solo se enchufan.

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

![Diagrama de cableado: adaptador de 5V al divisor R1/R2/C1 hacia A0 de la NodeMCU; un controlador de carga (AC → 12V) mantiene cargada una batería de 12V independiente, que alimenta, a través de un regulador step-down, a la NodeMCU y la lógica del relé, y directamente al contacto COM del relé, que conmuta 12V hacia la luz de emergencia; GND común entre todos los bloques.](cableado.svg)

*El adaptador de 5V solo alimenta el divisor de voltaje (el "sensor"). El
controlador de carga toma AC de la misma toma monitoreada y mantiene
cargada la batería de 12V — de ahí sale toda la energía del resto del
sistema: la NodeMCU y la lógica del relé a través de un regulador
step-down a 5V, y la luz directamente en 12V a través del contacto del
relé. El relé y la NodeMCU comparten GND, y la NodeMCU controla el relé
con la señal digital `D2` (`GPIO4`).*

## Divisor de voltaje (detección de corriente)

Componentes: **R1 = 10 kΩ**, **R2 = 15 kΩ**, **C1 = 0.1 µF**, alimentados
desde la salida de 5V del **adaptador** enchufado en la toma monitoreada
(ver el detalle de R1/R2/C1 en el diagrama de arriba).

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
- `VCC` del relé → salida de 5V del **regulador step-down** (ver más abajo).
- `GND` del relé → `GND` común con la NodeMCU y la batería.
- `COM` del relé → **+12V directo de la batería** (sin pasar por el
  regulador — el contacto conmuta la tensión completa de la batería).
- `NO` del relé → terminal **+** de la luz de emergencia. El terminal **–**
  de la luz vuelve al `GND` común (a la batería).
- Revisa si tu módulo de relé es activo en bajo (la mayoría de los módulos
  de 1 canal lo son). Si el relé enciende la luz al revés de lo esperado
  (encendida cuando debería estar apagada), cambia `inverted: true` en el
  `switch` `relay0` del YAML.

## Regulador 5V (step-down 12V → 5V)

La NodeMCU y la lógica del relé trabajan a 5V, pero la batería de respaldo
es de 12V. Un regulador step-down (buck) tipo LM2596 o similar convierte
ese 12V a un 5V estable:

- Entrada: `+12V` y `GND` de la batería de respaldo.
- Salida: `5V` y `GND` hacia `VIN`/`GND` de la NodeMCU y `VCC`/`GND` del
  relé.
- Elegí un módulo que entregue al menos 500 mA — la NodeMCU y un relé
  consumen poco, pero conviene dejar margen.
- El contacto `COM`/`NO` del relé **no** pasa por este regulador: se
  alimenta directo de los 12V de la batería, ya que la luz necesita la
  tensión completa.

## Controlador de carga (AC → 12V)

La batería de respaldo necesita mantenerse cargada mientras hay corriente,
para estar lista cuando ocurra un corte. Un **controlador de carga** —un
módulo sellado que se enchufa directo a AC, como los que se usan para
baterías de luces de emergencia o alarmas— hace ese trabajo:

- Se enchufa en la **misma toma que estás monitoreando** con el adaptador
  de 5V: mientras haya corriente, mantiene la batería cargada (con corte
  automático o carga flotante, según el modelo, para no sobrecargarla).
- Su salida `+12V`/`GND` va directo a la batería (mismo terminal que usan
  el regulador y el `COM` del relé — ver el diagrama).
- Durante un corte, el controlador también pierde alimentación y deja de
  cargar, pero para entonces la batería ya debería estar cargada y es ella
  la que sostiene todo el sistema — por eso importa que sea de **buena
  capacidad** y esté en buen estado (ver más abajo).
- No cablees el lado de AC vos mismo: comprá un controlador/cargador de 12V
  ya armado y enchufalo como a cualquier otro adaptador de pared.

## Alimentación de respaldo — requisito de diseño

La NodeMCU y el relé necesitan estar **energizados durante el corte** para
poder encender la luz en el momento en que ocurre. Si los alimentas
únicamente desde el mismo adaptador de 5V que estás monitoreando, el corte
de corriente también los apaga justo cuando deben actuar, y el sistema no
podrá encender el relé.

Por eso todo el sistema —NodeMCU, relé y la luz de emergencia misma— se
alimenta de una **batería de respaldo de 12V**, independiente del
adaptador monitoreado. El adaptador de 5V solo alimenta el lado de
**medición** (el divisor de voltaje), nunca la NodeMCU ni la luz.

## Checklist de armado

1. Arma el divisor R1/R2/C1 en una zona de baja tensión.
2. Enchufa el adaptador de 5V en la toma que quieres monitorear y, antes de
   conectar nada a la NodeMCU, mide con multímetro el voltaje en el nodo del
   divisor — confirma que sea ~3V y nunca mayor a 3.3V.
3. Conecta `A0` y `GND` del divisor a la NodeMCU.
4. Conecta el controlador de carga a la batería (`+12V`/`GND`) y enchufalo
   en la misma toma monitoreada. Con la batería conectada, medí con
   multímetro que esté cargando (voltaje en sus bornes cercano o igual al
   nominal de la batería).
5. Cablea el regulador step-down: entrada `+12V`/`GND` a la batería, salida
   `5V`/`GND` a `VIN`/`GND` de la NodeMCU y `VCC`/`GND` del relé. Antes de
   conectar la NodeMCU, medí con multímetro que la salida sea ~5V.
6. Cablea el relé: `IN` a `D2`, `COM` directo a `+12V` de la batería, `NO` a
   la entrada `+` de la luz de emergencia, y la `–` de la luz de vuelta al
   `GND` común.
7. Cierra el gabinete manteniendo el divisor, el controlador de carga, el
   regulador y el relé ordenados y protegidos de contacto accidental.
8. Energiza todo y verifica en Home Assistant que "Electricidad" refleje el
   estado real, y que al provocar un corte (desenchufando el adaptador de
   5V monitoreado) el relé encienda tras el retardo configurado.
