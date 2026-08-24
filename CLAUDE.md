# AmandaGestion

Sistema interno de gestión de **Amanda Gráficos** (productos personalizados: stickers,
vinilos, UV, tarjetas, Party Box, banners, etc.). Lo usamos Eli y Sofía para administrar
producción, costos y cobros. Desplegado en `gestion.amandagraficos.store`.

> No es la web pública (esa es `amandagraficos.store`, solo para clientes). Esto es la
> herramienta interna.

---

## Stack técnico

- **Frontend:** un único archivo `index.html` con HTML/CSS/JS plano. **Sin frameworks**,
  sin build step.
- **Backend / datos:** Supabase (cliente `sb`). Seguridad por RLS.
- **Login:** compartido entre Eli y Sofi (no hay usuarios separados).
- **Idioma:** todo en español rioplatense (UI, comentarios, mensajes).

---

## Reglas de arquitectura (NO romper)

1. **Todo en un solo `index.html`.** No partir en módulos ni meter un bundler.
2. **Cero frameworks y cero build.** Si hace falta una librería, traerla por CDN y
   justificarlo.
3. **Solo la `anon key` pública de Supabase en el front.** Nunca la `service_role`.
   La seguridad se apoya en RLS.
4. **Antes de tocar el esquema de Supabase (tablas/columnas), avisar y proponer el
   cambio.** No crear ni borrar columnas sin confirmar.
5. **Cambios chicos y verificables.** Editar lo justo y explicar qué se tocó.

---

## Tablas en Supabase

- `pedidos` — la principal (ver campos abajo).
- `clientes` — id, nombre, whatsapp, tipo ('particular', etc.). Si se carga un pedido sin
  cliente vinculado, se crea uno automáticamente.
- `gastos` — gastos del negocio (monto, fecha, pagado_por, puede vincularse a `stock`).
- `stock` — insumos (nombre, etc.).
- `envios_plotter` — registro de envíos al plotter.
- `tarifa_config` — config única (id = 1): `valor_hora`, `margen` (margen objetivo en %),
  `materiales` (objeto con precio por m² según tipo de material).

---

## Campos de la tabla `pedidos`

Datos del cliente / pedido:
- `id`
- `cliente_nombre`, `cliente_whatsapp`, `cliente_id`
- `descripcion` — obligatoria
- `estado` — una de las 8 etapas (ver abajo)
- `precio_cliente` — lo que se le cobra al cliente
- `cobrado` (boolean) y `cobrado_por` (Eli/Sofi)
- `notas`
- `cargado_por` — Eli o Sofi (quién lo cargó)
- `origen`, `tipo_producto`

Datos para costo y producción:
- `cantidad`, `ancho_cm`, `alto_cm`
- `plotter_usado` — texto (qué plotter se usó), NO es un costo
- `material_tipo` — tipo de material (se cruza con `tarifa_config.materiales`)
- `minutos_diseno`, `minutos_corte` — tiempos de trabajo
- `costo_materiales` — costo de materiales (manual o autocalculado)
- `costo_mo` — costo de mano de obra (autocalculado, ver abajo)

---

## Estados del pipeline (campo `estado`) — son 8

1. `recibido` — Recibido
2. `diseno` — En diseño
3. `plotter` — Plotter (enviado al plotter)
4. `retirar` — Listo para retirar
5. `preparacion` — En preparación
6. `cortar` — Falta cortar
7. `listo` — Listo para entregar
8. `entregado` — Entregado

(La lista está definida en la constante `ESTADOS`, con su color cada una.)

---

## Costos y margen (¡OJO, esto cambió respecto a la idea original!)

El costo de un pedido **NO es un solo campo**. Se compone de:

- **Materiales** (`costo_materiales`): se puede cargar a mano, o el sistema lo calcula
  solo como `área (m²) × precio por m²` del tipo de material, usando `ancho_cm`,
  `alto_cm`, `cantidad` y la tabla `tarifa_config.materiales`.
- **Mano de obra** (`costo_mo`): se autocalcula como
  `((minutos_diseno + minutos_corte) / 60) × valor_hora`, con el `valor_hora` de
  `tarifa_config`.

Cálculos en la pantalla de tarifas:
- `ganancia = precio_cliente − costo_materiales − costo_mo`
- `costo_total = costo_materiales + costo_mo`
- `precio_sugerido = costo_total × (1 + margen / 100)`  (margen objetivo de la config)

**Importante:** hoy la ganancia **SÍ incluye la mano de obra**. (Esto contradice una nota
vieja que decía que no se contaba; el código sí la cuenta.) No cambiar este criterio sin
que Eli lo pida explícitamente.

Los cotizadores (vinilos, UV, tarjetas) muestran ganancia en pesos. La ganancia se ve en
pesos y el margen objetivo en %.

---

## Tablero (dashboard)

Métricas principales:
- **Activos** — pedidos con `estado` distinto de `entregado`.
- **A cobrar** — suma de `precio_cliente` de los pedidos con `cobrado = false`.
- **Cobrado** — suma de `precio_cliente` de los pedidos con `cobrado = true`.
- **Saldo** — `cobrado − total de gastos` (verde si positivo, rojo si negativo).

Otras: trabajos entregados, unidades de stickers, m² producidos, y un **balance por
persona** (Eli / Sofi) que cruza cobros y gastos de cada uno.

---

## Cotizadores

Hay herramientas de cotización para: vinilos (V), vinilos UV, y tarjetas. Calculan área,
costo, venta y ganancia por pieza, con un margen objetivo configurable.

---

## Flujo de trabajo

- **Prueba local:** se trabaja y se prueba localmente en VS Code antes de subir nada.
- **Despliegue:** cuando está OK, `git push` → sube a GitHub → se publica solo en
  `gestion.amandagraficos.store`.
- Regla: **no pushear sin probar local primero.**

---

## Convenciones

- Textos y etiquetas de la UI en español rioplatense, tono cercano.
- Nombres de columnas en Supabase en snake_case y en español (`cliente_nombre`,
  `costo_materiales`, etc.). Mantener ese estilo.
- Evitar combinaciones de azul + amarillo en lo visual (preferencia de marca).

---

## Notas

- Mantener este archivo por debajo de ~200 líneas.
- Revisarlo cuando cambie algo importante del proyecto.
