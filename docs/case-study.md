# Caso de uso: una jornada de punta a punta

> **⚠ DATOS FICTICIOS.** Este caso es una simulación ilustrativa del proceso. Los activos ("ALFA", "BETA"), los precios, las cantidades, los códigos y los horarios son inventados a los fines del ejemplo. Ninguna cifra de este documento corresponde a una posición u operación real. El documento de arquitectura está en [`architecture.svg`](architecture.svg); los conceptos, en el [README](../README.md).

Este caso sigue **una jornada de operación** a través del pipeline completo, con **dos hilos en paralelo** elegidos a propósito: uno termina en **negación** (hilo ALFA) y el otro recorre el camino completo hasta el registro y las métricas (hilo BETA). Ambos son outputs del mismo proceso — ese es el punto central del diseño.

---

## Etapa 1 — Detección y ruteo (pasada de apertura)

La primera pasada del día recorre el roster completo — cada posición bajo su propia hipótesis — precedida por el paso 0: el pulso del conjunto, leído del dashboard.

**Artefacto: acta de pasada** *(ficticio)*

```
PASADA 1/3 — apertura                                      [jornada D]
paso 0 · pulso del conjunto: sin banderas de régimen; dispersión normal

hallazgos:
  H-1  ALFA   −4,8 % vs. sector plano, sin noticia asociada
       → evaluación (puerta activo)
  H-2  BETA   precio a 1,2 % de la condición de entrada del plan vigente
       → evaluación (puerta activo, revalidación corta)

resto del roster (14 posiciones): sin hallazgos — pasada registrada
```

Dos cosas para notar: la pasada **no evalúa** (detecta y rutea; los juicios vienen después), y el "sin hallazgos" de las otras 14 posiciones queda registrado — la cobertura completa es verificable, no supuesta.

---

## Etapa 2 — Evaluación

### Hilo ALFA → negación

El hallazgo H-1 entra por la puerta activo. La pregunta implícita —"¿aprovechamos la caída?"— se somete a la refutación de tres frentes.

**Artefacto: acta de evaluación** *(ficticio)*

```
EVALUACIÓN — ALFA (puerta: activo, disparador: H-1)

hipótesis en la ficha viva:        ninguna vigente sobre este movimiento
hipótesis implícita del hallazgo:  "comprar la caída"

REFUTAR:
  realidad    caída confirmada (−4,8 %); sin noticia, sin evento sectorial
  causalidad  ✗ no hay mecanismo declarado que conecte el precio de hoy
              con una tesis: la única premisa disponible ES el precio
  adversario  quien vende hoy puede saber algo que este análisis no ve;
              sin hipótesis propia no hay forma de discriminarlo

TERMINAL: negación{causa: persecución invertida — movimiento sin
          hipótesis; el precio no es tesis}

siembra → backlog: "investigar causa del desacople de ALFA
                    si persiste ≥ 3 jornadas"
```

La negación **no descarta el interés**: lo transforma en un pendiente con condición ("si persiste"), registrado donde corresponde. Lo que bloquea es operar sin hipótesis.

### Hilo BETA → tripleta

El hallazgo H-2 es distinto: BETA tiene una hipótesis viva en su ficha, con derrotadores declarados. La evaluación aquí es una **revalidación corta**: ¿algún derrotador disparó desde la última revisión?

**Artefacto: acta de evaluación** *(ficticio)*

```
EVALUACIÓN — BETA (puerta: activo, disparador: H-2)

hipótesis viva:   HIP-B03 "la normalización de márgenes del sector
                  debería reflejarse antes en los especialistas que
                  en los integrados"
derrotadores:     (a) deterioro de márgenes 2 trimestres seguidos → NO disparado
                  (b) cambio regulatorio sobre el segmento          → NO disparado

TERMINAL: tripleta{BETA, HIP-B03, acumular un tramo}
```

**Gate del operador (¿actuar?):** la tripleta se presenta al operador, que decide actuar. Sin este "sí" humano explícito, el pipeline se detiene aquí.

---

## Etapa 3 — Planificación de ejecución

La tripleta se convierte en una acción condicionada de cuatro componentes. El cuarto —el límite— es obligatorio.

**Artefacto: especificación de acción** *(ficticio)*

```
ACCIÓN CONDICIONADA — BETA / HIP-B03

ACCIÓN      comprar 40 unidades (≈ 500 u.m.)
PLAZO       esta jornada y la siguiente
CONDICIÓN   precio ≤ 12,50 (zona de la premisa) y libro con
            profundidad normal
LÍMITE      si en 2 jornadas la condición no se da, la acción se
            libera sin ejecutarse y BETA vuelve a vigilancia
```

Si el límite venciera, la liberación quedaría registrada como negación tipada — el mismo destino documental que cualquier otro resultado.

---

## Etapa 4 — Ejecución (gate humano)

A media jornada, la condición se cumple. El sistema arma la orden completa y la presenta; **la colocación es manual, del operador**.

**Artefacto: orden lista para colocar** *(ficticio)*

```
ORDEN LISTA — colocación manual del operador

instrumento   BETA
operación     compra
cantidad      40 unidades
precio        límite 12,48
chequeos      libro con profundidad normal ✓
              sin órdenes duplicadas en proceso ✓
```

El operador coloca la orden en su plataforma. Fill confirmado: **40 unidades @ 12,46, a las 11:32**. La verificación posterior se hace contra el registro del broker, no contra lo que se cree haber pedido.

---

## Etapa 5 — Registro

El asiento captura **en el momento** lo que después sería irrecuperable: la atribución del fill a su lote funcional y tramo.

**Artefacto: asiento de ledger** *(ficticio)*

```
ASIENTO — ledger (append-only)

fill          11:32 · BETA · +40 @ 12,46                    [verificado]
atribución    lote L-07
              mecanismo: acumulación por tesis
              horizonte: 6–12 meses
              criterio de éxito: cierre del descuento vs. premisa
tramo         núcleo
patas         referencia externa 12,44 [verificado] · ejecutado 12,46
              spread de ejecución +0,16 %
invariante    Σ lotes BETA en ledger (160) = posición broker (160)  ✓
```

Nótese el estatuto de cada dato (`verificado`) y la verificación del invariante en el mismo asiento: la consistencia del ledger no se audita "cada tanto" — se verifica en cada escritura.

---

## Etapa 6 — Métricas derivadas

El asiento es un evento; el dashboard reacciona.

**Artefacto: extracto de dashboard** *(ficticio)*

```
DASHBOARD — regeneración incremental (evento #12 de la jornada)

BETA · lote L-07
  costo del lote      1.994 → 2.492 u.m.
  no-realizado        +1,1 %
  vs. vara de cluster +0,4 pp

conjunto
  negaciones de la jornada: 1 (ALFA — persecución invertida)
  pasadas completadas: 2/3 · roster cubierto: 16/16
```

Al cierre de la jornada, el disparador de reloj fuerza además una **regeneración total** — la foto de fin de día existe aunque nadie la pida.

---

## Cierre de sesión — reconciliación

Todo documento tocado gana su línea de LOG; los invariantes se verifican; lo resuelto se borra de la agenda.

**Artefacto: líneas de LOG** *(ficticio)*

```
LOG (append)
[D] ficha BETA    | tramo núcleo +40 vía lote L-07; plan sin cambios;
                  | acción condicionada cumplida dentro del límite
[D] evaluación    | negación ALFA asentada (persecución invertida);
                  | semilla a backlog con condición de persistencia
[D] dashboard     | regeneración total de cierre ✓ · invariantes ✓
```

---

## Qué muestra este caso

1. **Las dos terminales son outputs del mismo proceso.** La negación de ALFA y la ejecución de BETA recorren las mismas etapas, dejan artefactos del mismo tipo y terminan en el mismo sistema documental. Un proceso que solo registra sus "síes" no puede medir su propia selectividad.

2. **Cada etapa consume el contrato de la anterior y nada más.** El acta de pasada no contiene juicios; el acta de evaluación no contiene tamaños; la especificación de acción no contiene la orden; la orden no contiene la atribución. Cuando una etapa necesita algo que su contrato no trae, eso es una señal de diseño.

3. **Los gates humanos son visibles y auditables.** Hay exactamente dos "síes" del operador en el camino de BETA (actuar; colocar), y ambos quedan implícitos en los artefactos que los siguen.

4. **La información perecedera se capturó a tiempo.** La atribución del fill al lote L-07 y al tramo núcleo existe porque el asiento la exigió a las 11:32. Mañana, ese dato ya no estaría en ninguna parte.

5. **Trazabilidad completa decisión → registro → métrica.** Desde H-2 hasta la línea de LOG hay una cadena de artefactos tipados que un tercero puede auditar sin haber estado presente.

---

*⚠ Recordatorio: todos los datos de este caso son ficticios. Documento público — no contiene posiciones, montos, plataformas ni operaciones reales.*
