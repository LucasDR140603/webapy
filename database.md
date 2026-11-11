# Base de datos de control horario: contexto para generar consultas SQL

Contexto para un modelo que traduce preguntas en lenguaje natural a **una única consulta `SELECT` de PostgreSQL**. Este documento es la fuente de verdad del esquema: no existen otras tablas, vistas ni columnas.

- Describe la **estructura**, las **reglas de negocio** y los **patrones de consulta**. **No contiene datos**: la base de destino tiene esta misma estructura, pero sus clientes, centros, proyectos, usuarios y registros son otros. No asumas que existe ningún nombre ni valor concreto.
- Estructura extraída el 2026-11-11 de `information_schema` / `pg_catalog` de una copia idéntica. Los ejemplos de la sección 10 se validaron sobre esa copia.

| Aspecto | Qué asumir |
|---|---|
| Motor | PostgreSQL. El SQL de este documento es compatible con la versión 10 o superior |
| Esquema | `public` (no hace falta prefijar las tablas) |
| Fechas y horas | `timestamp without time zone`, en la hora local con la que escribe la aplicación. "Ahora" = `LOCALTIMESTAMP`; "hoy" = `CURRENT_DATE` |
| Formato de fecha del servidor (`DateStyle`) | Desconocido: escribe siempre literales `DATE 'AAAA-MM-DD'` y formatea con `to_char()` |
| Idioma del servidor (`lc_time`) | Desconocido: no uses `TMDay` ni `TMMonth`. Los nombres de días y meses en español salen de los arrays de la sección 6.4 |
| Extensiones | No asumas ninguna (tampoco `unaccent`) |
| Vistas, enums, particiones | Ninguna |

---

## 1. Dominio

Aplicación de **control horario / partes de trabajo**.

- Cada trabajador (`usuarios`) inicia y detiene **tramos de tiempo** (`registros`) imputados a un proyecto.
- Cada proyecto (`proyectos`) pertenece a un **centro** (`centros`): la planta, fábrica, instalación o sede donde se trabaja.
- Cada centro pertenece a un **cliente** (`clientes`).
- Un registro puede llevar gastos: kilómetros, desplazamiento, manutención y alojamiento.

`registros` es la tabla de hechos: casi todas las preguntas (horas, gastos, actividad) se responden agregándola.

---

## 2. Esquema resumido

```text
clientes  (id PK, nombre UQ, abierto)
centros   (id PK, nombre UQ, id_cliente FK→clientes.id, abierto)
proyectos (id PK, nombre UQ, id_centro FK→centros.id, descripcion, inicio, fin)
registros (id PK, id_proyecto FK→proyectos.id, inicio, fin, descripcion, observaciones,
           id_usuario FK→usuarios.id, km, desplazamiento, manutencion, alojamiento)
usuarios  (id PK, nombre, apellido1, apellido2, nick UQ, clave ⛔, email UQ, telefono UQ,
           administrador, operativo, perfil)

clientes 1──N centros 1──N proyectos 1──N registros N──1 usuarios
```

Cadena de JOIN canónica (usa siempre estos alias):

```sql
FROM registros r
JOIN proyectos p  ON p.id  = r.id_proyecto
JOIN centros   ce ON ce.id = p.id_centro
JOIN clientes  cl ON cl.id = ce.id_cliente
JOIN usuarios  u  ON u.id  = r.id_usuario
```

- Todas las FK son `NOT NULL` y `ON DELETE CASCADE`: no hay filas huérfanas y los `INNER JOIN` desde `registros` nunca pierden filas.
- Usuarios y proyectos/centros/clientes solo se relacionan a través de `registros`.
- `id`, `nombre`, `descripcion`, `inicio`, `fin` y `abierto` se repiten en varias tablas: **cualifica siempre cada columna con su alias**.
- Todas las PK son `uuid`; no hay ids numéricos ni secuencias.

---

## 3. Reglas para generar la consulta

1. **Solo lectura.** Una única sentencia `SELECT` (vale `WITH … SELECT`). Nunca `INSERT`, `UPDATE`, `DELETE`, DDL, `SET` ni varias sentencias.
2. **No inventes** tablas, columnas ni valores. No existen tarifas, precios por hora, facturas, nóminas, vacaciones ni un campo "facturable".
3. **`usuarios.clave` está prohibida** en cualquier cláusula (`SELECT`, `WHERE`, `JOIN`, `ORDER BY`…). Nunca `SELECT *` ni `u.*` sobre `usuarios`. `email` y `telefono` solo si se piden expresamente.
4. **Resultados legibles.** Devuelve nombres (cliente, centro, proyecto, `nick`), no UUID, salvo que se pidan. Incluye las columnas que identifican cada fila del resultado.
5. **Duración** = `EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600`: horas decimales, contando el tramo en curso hasta ahora. **Convierte a `numeric` antes de redondear**: `ROUND(SUM(…)::numeric, 2)` o `ROUND((…)::numeric, 2)`. Antes de PostgreSQL 14, `EXTRACT` devuelve `double precision` y `ROUND(x, 2)` falla sin la conversión.
6. **Fechas.** Filtra y agrupa por `r.inicio` (cada tramo cuenta en el día en que empieza). Usa intervalos semiabiertos (`>= desde AND < hasta`) y literales `DATE 'AAAA-MM-DD'`; nunca `'01/02/2026'`, cuyo significado depende del servidor. No uses `BETWEEN` con timestamps. Si la pregunta no indica periodo, no filtres por fecha (todo el histórico).
7. **Búsqueda de texto y nombres.** Si tienes el nombre exacto, usa `=`. Si no, búsqueda parcial: `ILIKE '%texto%'` ignora mayúsculas pero **no tildes**. Para ignorar ambas: `translate(lower(col), 'áéíóúüñÁÉÍÓÚÜÑ', 'aeiouunaeiouun') LIKE '%texto%'`, con el patrón en minúsculas y sin tildes. Sigue la sección 8 para resolver nombres.
8. **Personas.** Identifícalas por `u.nick` o por nombre y apellidos (`u.nick ILIKE '%ana%' OR u.nombre ILIKE '%ana%' OR u.apellido1 ILIKE '%ana%'`). Agrupa por `u.id` además del nombre, porque `nombre` no es único, y devuelve `u.nick` para distinguir coincidencias.
9. **"Activo / abierto"** depende de la tabla: `clientes.abierto`, `centros.abierto`, `proyectos.fin IS NULL`, `usuarios.operativo`; registro en curso = `registros.fin IS NULL` (sección 6.3).
10. **Entidades sin actividad o con cero horas:** parte de la tabla padre con `LEFT JOIN registros` y pon los filtros de `registros` (fechas, etc.) en el `ON`, no en el `WHERE`; o usa `NOT EXISTS`.
11. **Alias.** En `WHERE` y `HAVING` no se pueden usar alias del `SELECT`: repite la expresión. En `ORDER BY` sí. Nombra las columnas de salida en español y `snake_case` (`horas`, `cliente`, `proyecto`, `dia`, `km`).
12. **Listados sin agregar:** `ORDER BY` explícito y `LIMIT 100` salvo que se pidan todos. Si formateas fechas con `to_char`, ordena por la columna original (`ORDER BY r.inicio`), no por el texto.
13. **Nombres de días y meses** en español: usa los arrays de la sección 6.4, no `to_char(…, 'TMDay')` ni `'TMMonth'`.
14. **Cero filas es una respuesta válida.** No relajes ni cambies los filtros para forzar resultados, y no excluyas registros por parecer anómalos (sección 9) salvo que se pida.

---

## 4. Tablas

### 4.1 `clientes`: empresas cliente

| Columna | Tipo | Nulo | Default | Significado |
|---|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` | PK |
| `nombre` | varchar(100) | no | | Nombre del cliente. UNIQUE. Mayúsculas y tildes pueden variar: compara con `ILIKE` o `translate` |
| `abierto` | boolean | no | `true` | `true` = cliente activo. Al cerrarlo se cierran en cascada sus centros (y con ellos proyectos y registros en curso) |

### 4.2 `centros`: plantas, instalaciones o sedes de un cliente

| Columna | Tipo | Nulo | Default | Significado |
|---|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` | PK |
| `nombre` | varchar(100) | no | | Nombre del centro (a menudo una localidad o el nombre de la planta). UNIQUE en toda la tabla, no por cliente. Puede tener espacios sobrantes: evita `=` con texto aproximado |
| `id_cliente` | uuid | no | | FK → `clientes.id` |
| `abierto` | boolean | no | `true` | `true` = centro activo. Al cerrarlo, sus proyectos activos reciben `fin` = ahora |

### 4.3 `proyectos`: trabajos o encargos que se hacen en un centro

| Columna | Tipo | Nulo | Default | Significado |
|---|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` | PK |
| `nombre` | varchar(100) | no | | Nombre del proyecto. UNIQUE en toda la tabla |
| `id_centro` | uuid | no | | FK → `centros.id` |
| `descripcion` | text | sí | | Descripción libre; útil para buscar por tema o tecnología |
| `inicio` | timestamp | no | `date_trunc('minute', LOCALTIMESTAMP)` | Fecha de **alta** del proyecto en el sistema. Puede no coincidir con el comienzo real del trabajo (p. ej. en cargas iniciales). Para "cuándo se empezó a trabajar" usa `MIN(r.inicio)` |
| `fin` | timestamp | sí | | Fecha de finalización. **`NULL` = proyecto activo.** Un proyecto finalizado no admite registros nuevos |

### 4.4 `registros`: tramos de tiempo trabajado (tabla de hechos)

Cada fila es un tramo continuo de un usuario en un proyecto. Un día de trabajo puede tener varios tramos, en uno o varios proyectos.

| Columna | Tipo | Nulo | Default | Significado |
|---|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` | PK |
| `id_proyecto` | uuid | no | | FK → `proyectos.id` |
| `inicio` | timestamp | no | `date_trunc('minute', LOCALTIMESTAMP)` | Comienzo del tramo, en hora local y normalmente con precisión de minuto |
| `fin` | timestamp | sí | | Final del tramo. **`NULL` = registro en curso** (la persona está trabajando ahora). Como máximo hay uno en curso por usuario |
| `descripcion` | text | sí | | Qué se hizo. Opcional y a menudo vacía |
| `observaciones` | text | sí | | Notas libres, normalmente sobre gastos (kilómetros, peajes, comidas). Opcional y casi siempre vacía |
| `id_usuario` | uuid | no | | FK → `usuarios.id` |
| `km` | numeric(10,2) | no | `0` | Kilómetros recorridos (no son euros) |
| `desplazamiento` | numeric(10,2) | no | `0` | Gasto de desplazamiento (peajes, transporte), en € |
| `manutencion` | numeric(10,2) | no | `0` | Gasto de manutención o dietas (comidas), en € |
| `alojamiento` | numeric(10,2) | no | `0` | Gasto de alojamiento (hotel), en € |

- No hay columna de duración: se calcula (regla 5 y sección 6.1).
- Los cuatro campos de gasto nunca son NULL (default 0): "registros con gasto" = `col > 0`.
- **La unidad de los gastos puede no ser homogénea.** Los anota cada usuario: en la copia de referencia unos ponían importes reales y otro un `1` como marca de "hubo gasto" (1 desplazamiento, 1 comida). `SUM()` mezclaría ambas cosas; para contar días o registros con gasto usa `COUNT(*) FILTER (WHERE col > 0)`.

### 4.5 `usuarios`: trabajadores que registran tiempo

| Columna | Tipo | Nulo | Default | Significado |
|---|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` | PK |
| `nombre` | varchar(100) | no | | Nombre de pila (no único) |
| `apellido1` | varchar(150) | no | | Primer apellido |
| `apellido2` | varchar(150) | sí | | Segundo apellido |
| `nick` | varchar(50) | no | | Alias de acceso. UNIQUE: el mejor identificador legible |
| `clave` | text | no | | ⛔ Credencial de acceso. **Nunca la consultes ni la devuelvas** |
| `email` | varchar(150) | no | | UNIQUE. Dato personal: solo si se pide expresamente |
| `telefono` | varchar(20) | no | | UNIQUE. Dato personal: solo si se pide expresamente |
| `administrador` | boolean | no | `false` | Tiene permisos de administración en la aplicación |
| `operativo` | boolean | no | `true` | `true` = usuario activo. Uno no operativo no puede crear registros; al desactivarlo se cierran sus registros en curso |
| `perfil` | varchar(255) | sí | | Nombre del archivo de la foto de perfil. No es un rol ni un puesto |

Nombre completo:

```sql
u.nombre || ' ' || u.apellido1 || COALESCE(' ' || u.apellido2, '') AS nombre_completo
```

---

## 5. Reglas de negocio (triggers) y lo que implican

| Tabla | Momento | Regla |
|---|---|---|
| `registros` | antes de INSERT | Rechaza si el usuario no está operativo o el proyecto tiene `fin`; exige `inicio <= fin`. Si el nuevo registro empieza abierto (`fin` NULL), **cierra el registro abierto anterior del mismo usuario** con `fin` = `inicio` del nuevo |
| `registros` | antes de UPDATE | No se puede cambiar el `id` ni reabrir un registro cerrado; exige `inicio <= fin` |
| `proyectos` | antes de INSERT | No se puede crear en un centro cerrado; exige `inicio <= fin` |
| `proyectos` | antes de UPDATE | Al informar `fin`, cierra (fin = ahora) los registros en curso del proyecto; no se puede reabrir si el centro está cerrado |
| `centros` | antes de INSERT / UPDATE | No se puede crear ni reabrir con el cliente cerrado; al cerrarlo pone `fin` = ahora a sus proyectos activos |
| `clientes` | antes de UPDATE | Al cerrarlo cierra todos sus centros (y en cascada proyectos y registros en curso) |
| `usuarios` | antes de UPDATE | Al desactivarlo (`operativo = false`) cierra sus registros en curso |
| todas | después de INSERT / UPDATE / DELETE | `pg_notify` a los canales `cambios` y `notification`. No afecta a las consultas |

Consecuencias para las consultas:

- Hay **como máximo un registro en curso por usuario**. "¿Quién está trabajando ahora?" = `WHERE r.fin IS NULL`.
- Si alguien olvida parar, su tramo dura hasta que inicia el siguiente: pueden aparecer registros de muchas horas o de varios días (sección 9).
- El cierre baja en cascada: un proyecto activo (`fin IS NULL`) siempre está en un centro abierto de un cliente abierto. Para "proyectos activos" basta `p.fin IS NULL`.
- No hay borrado lógico aparte de estos indicadores: lo que se borra desaparece con todos sus hijos.

---

## 6. Cálculos y expresiones estándar

### 6.1 Horas

```sql
-- horas de un tramo (incluye el tramo en curso hasta ahora)
EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600

-- horas de un tramo, redondeadas
ROUND((EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600)::numeric, 2) AS horas

-- total redondeado
ROUND(SUM(EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600)::numeric, 2) AS horas

-- total que debe valer 0 cuando no hay registros (LEFT JOIN)
ROUND(COALESCE(SUM(EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600), 0)::numeric, 2) AS horas

-- solo tramos cerrados (si se pide expresamente)
WHERE r.fin IS NOT NULL

-- formato h:mm a partir de un total en segundos s
floor(s / 3600)::int || ':' || lpad((floor(s / 60)::int % 60)::text, 2, '0')
```

No uses `to_char` sobre intervalos para mostrar totales: pierde los días (`'2 days 03:00'` → `03:00`).

### 6.2 Fechas relativas (sobre `r.inicio`)

| Expresión | Condición |
|---|---|
| hoy | `r.inicio >= CURRENT_DATE AND r.inicio < CURRENT_DATE + 1` |
| ayer | `r.inicio >= CURRENT_DATE - 1 AND r.inicio < CURRENT_DATE` |
| esta semana (lunes a domingo) | `r.inicio >= date_trunc('week', LOCALTIMESTAMP) AND r.inicio < date_trunc('week', LOCALTIMESTAMP) + INTERVAL '1 week'` |
| la semana pasada | `r.inicio >= date_trunc('week', LOCALTIMESTAMP) - INTERVAL '1 week' AND r.inicio < date_trunc('week', LOCALTIMESTAMP)` |
| este mes | `r.inicio >= date_trunc('month', LOCALTIMESTAMP) AND r.inicio < date_trunc('month', LOCALTIMESTAMP) + INTERVAL '1 month'` |
| el mes pasado | `r.inicio >= date_trunc('month', LOCALTIMESTAMP) - INTERVAL '1 month' AND r.inicio < date_trunc('month', LOCALTIMESTAMP)` |
| este trimestre | `r.inicio >= date_trunc('quarter', LOCALTIMESTAMP) AND r.inicio < date_trunc('quarter', LOCALTIMESTAMP) + INTERVAL '3 months'` |
| este año | `r.inicio >= date_trunc('year', LOCALTIMESTAMP) AND r.inicio < date_trunc('year', LOCALTIMESTAMP) + INTERVAL '1 year'` |
| el año pasado | `r.inicio >= date_trunc('year', LOCALTIMESTAMP) - INTERVAL '1 year' AND r.inicio < date_trunc('year', LOCALTIMESTAMP)` |
| últimos N días | `r.inicio >= LOCALTIMESTAMP - INTERVAL 'N days'` |
| en marzo de 2026 | `r.inicio >= DATE '2026-03-01' AND r.inicio < DATE '2026-04-01'` |
| el 5 de febrero de 2026 | `r.inicio >= DATE '2026-02-05' AND r.inicio < DATE '2026-02-06'` |
| un mes sin año ("en marzo") | Año en curso; si ese mes aún no ha llegado, el año anterior |
| fin de semana | `EXTRACT(ISODOW FROM r.inicio) IN (6, 7)` |
| días laborables | `EXTRACT(ISODOW FROM r.inicio) BETWEEN 1 AND 5` |

### 6.3 Estados

| Pregunta | Condición |
|---|---|
| clientes activos / cerrados | `cl.abierto` / `NOT cl.abierto` |
| centros activos / cerrados | `ce.abierto` / `NOT ce.abierto` |
| proyectos activos / finalizados | `p.fin IS NULL` / `p.fin IS NOT NULL` |
| usuarios activos / de baja | `u.operativo` / `NOT u.operativo` |
| administradores | `u.administrador` |
| registros en curso / cerrados | `r.fin IS NULL` / `r.fin IS NOT NULL` |

### 6.4 Agrupaciones y formatos

| Por | Expresión |
|---|---|
| día | `r.inicio::date` |
| semana (empieza en lunes) | `date_trunc('week', r.inicio)::date`, o `to_char(r.inicio, 'IYYY-"S"IW')` → `2026-S46` |
| mes | `to_char(date_trunc('month', r.inicio), 'YYYY-MM')` |
| año | `EXTRACT(YEAR FROM r.inicio)::int` |
| día de la semana (número) | `EXTRACT(ISODOW FROM r.inicio)` (1 = lunes … 7 = domingo) |
| hora del día | `EXTRACT(HOUR FROM r.inicio)` |
| fecha y hora legibles | `to_char(r.inicio, 'DD/MM/YYYY HH24:MI')` |

Nombres en español, independientes del idioma del servidor:

```sql
-- día de la semana
(ARRAY['lunes','martes','miércoles','jueves','viernes','sábado','domingo'])[EXTRACT(ISODOW FROM r.inicio)::int] AS dia_semana

-- mes ("marzo 2026")
(ARRAY['enero','febrero','marzo','abril','mayo','junio','julio','agosto','septiembre','octubre','noviembre','diciembre'])[EXTRACT(MONTH FROM r.inicio)::int]
  || ' ' || EXTRACT(YEAR FROM r.inicio)::int AS mes_texto
```

Jornada de un usuario en un día: entrada = `MIN(r.inicio)`, salida = `MAX(r.fin)`, horas = suma de sus tramos (puede haber huecos entre tramos).

---

## 7. Glosario: lenguaje natural → esquema

| El usuario dice… | Se refiere a |
|---|---|
| horas, tiempo, dedicación, "cuánto se ha trabajado" | suma de la duración de `registros` |
| registro, fichaje, parte, imputación, tramo, entrada | una fila de `registros` |
| jornada, día de trabajo | `registros` de un usuario agrupados por `r.inicio::date` |
| trabajador, empleado, técnico, persona, "quién" | `usuarios` |
| cliente, empresa, compañía | `clientes` |
| centro, planta, fábrica, instalación, sede, ubicación | `centros` |
| proyecto, obra, encargo, "trabajo para…" | `proyectos` |
| kilómetros, kilometraje | `r.km` |
| peajes, transporte, gastos de desplazamiento | `r.desplazamiento` |
| dietas, comidas, manutención | `r.manutencion` |
| hotel, alojamiento, noches fuera | `r.alojamiento` |
| gastos (en general) | `r.desplazamiento + r.manutencion + r.alojamiento` (en €); `km` aparte |
| trabajando ahora, fichado, en curso | `r.fin IS NULL` |
| proyecto terminado, cerrado, finalizado | `p.fin IS NOT NULL` |
| de baja, inactivo (persona) | `NOT u.operativo` |
| facturable, facturado, precio, coste por hora | No existe en el esquema (regla 2) |

---

## 8. Resolver nombres de clientes, centros, proyectos y personas

Los nombres solo existen en los datos: no hay catálogo fijo en este documento. Búscalos siempre en la propia consulta.

- **Tipo de entidad desconocido.** Si la pregunta nombra algo sin decir qué es ("horas de Acme"), puede ser un cliente, un centro o un proyecto: busca en los tres y devuelve las tres columnas.

  ```sql
  WHERE cl.nombre ILIKE '%acme%' OR ce.nombre ILIKE '%acme%' OR p.nombre ILIKE '%acme%'
  ```

- **Varias coincidencias.** Los nombres son UNIQUE en su tabla, pero un patrón parcial puede coincidir con varios. Un mismo texto puede ser a la vez nombre de cliente y de centro, o aparecer en centros de clientes distintos (p. ej. una localidad). Si la pregunta menciona también el cliente o el centro, filtra por ellos; si no, incluye `cliente` y `centro` en el resultado.
- **Variantes de escritura.** Mayúsculas, tildes, abreviaturas y espacios varían (`Administración`, `ADMINISTRACION`, `Admon`). Usa patrones cortos y sin tildes con `translate` (regla 7): `'%adm%'` cubre los tres; `'%administraci%'`, solo los dos primeros.
- **Temas o tecnologías** ("lo del PLC", "la aplicación de recetas"): busca también en `p.descripcion`, y en `r.descripcion` si se trata de trabajos concretos.
- **Personas:** `u.nick`, `u.nombre`, `u.apellido1` y `u.apellido2` (regla 8).
- **Catálogo opcional.** Si el mensaje incluye un bloque «Catálogo actual» con los nombres reales de la base, úsalo para elegir el nombre exacto y filtrar con `=`.

---

## 9. Calidad de los datos: qué puede aparecer

La estructura permite estas situaciones. No las excluyas salvo que se pida; sirven para preguntas de revisión o calidad.

- **Registros muy largos** (de un día para otro o de varios días): si alguien olvida parar, el tramo dura hasta que inicia el siguiente. Inflan los totales del periodo donde caen. Para revisarlos: `r.fin - r.inicio > INTERVAL '12 hours'`.
- **Registros de duración 0** (`r.fin = r.inicio`).
- **Registros solapados** del mismo usuario: al insertar tramos ya cerrados no se comprueba el solape, así que al sumar horas por usuario ese tiempo cuenta dos veces.
- **Registros que cruzan la medianoche:** cuentan enteros en el día de `r.inicio`.
- **Registros anteriores a `proyectos.inicio`:** no hay ninguna restricción que lo impida; no filtres registros por las fechas del proyecto.
- **Gastos en unidades mezcladas** (importes o marcas `1`; sección 4.4).
- **Textos vacíos** (`descripcion`, `observaciones`) y nombres con espacios sobrantes o escritura irregular.

Patrón para detectar solapes:

```sql
FROM registros a
JOIN registros b
  ON b.id_usuario = a.id_usuario
 AND a.id < b.id
 AND a.inicio < COALESCE(b.fin, LOCALTIMESTAMP)
 AND b.inicio < COALESCE(a.fin, LOCALTIMESTAMP)
```

---

## 10. Ejemplos (pregunta → SQL)

Validados sobre la copia de referencia (misma estructura). Los nombres (Acme, Marta, mantenimiento…) y las fechas son ilustrativos: sustitúyelos por los de la pregunta.

**1. ¿Cuántas horas se dedicaron a cada proyecto en enero de 2026?**

```sql
SELECT cl.nombre AS cliente,
       ce.nombre AS centro,
       p.nombre  AS proyecto,
       ROUND(SUM(EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600)::numeric, 2) AS horas
FROM registros r
JOIN proyectos p  ON p.id  = r.id_proyecto
JOIN centros   ce ON ce.id = p.id_centro
JOIN clientes  cl ON cl.id = ce.id_cliente
WHERE r.inicio >= DATE '2026-01-01'
  AND r.inicio <  DATE '2026-02-01'
GROUP BY cl.nombre, ce.nombre, p.nombre
ORDER BY horas DESC;
```

**2. ¿Cuántas horas ha trabajado cada persona esta semana?**

```sql
SELECT u.nick,
       u.nombre || ' ' || u.apellido1 AS usuario,
       ROUND(SUM(EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600)::numeric, 2) AS horas
FROM registros r
JOIN usuarios u ON u.id = r.id_usuario
WHERE r.inicio >= date_trunc('week', LOCALTIMESTAMP)
  AND r.inicio <  date_trunc('week', LOCALTIMESTAMP) + INTERVAL '1 week'
GROUP BY u.id, u.nick, u.nombre, u.apellido1
ORDER BY horas DESC;
```

**3. ¿Quién está trabajando ahora y en qué?**

```sql
SELECT u.nick,
       cl.nombre AS cliente,
       ce.nombre AS centro,
       p.nombre  AS proyecto,
       to_char(r.inicio, 'DD/MM/YYYY HH24:MI') AS desde,
       ROUND((EXTRACT(EPOCH FROM (LOCALTIMESTAMP - r.inicio)) / 3600)::numeric, 2) AS horas_transcurridas
FROM registros r
JOIN usuarios  u  ON u.id  = r.id_usuario
JOIN proyectos p  ON p.id  = r.id_proyecto
JOIN centros   ce ON ce.id = p.id_centro
JOIN clientes  cl ON cl.id = ce.id_cliente
WHERE r.fin IS NULL
ORDER BY r.inicio;
```

**4. Detalle de lo que se hizo en el proyecto de mantenimiento del cliente Acme en febrero de 2026.**

```sql
SELECT to_char(r.inicio, 'DD/MM/YYYY HH24:MI') AS desde,
       to_char(r.fin,    'DD/MM/YYYY HH24:MI') AS hasta,
       ROUND((EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600)::numeric, 2) AS horas,
       u.nick    AS usuario,
       ce.nombre AS centro,
       p.nombre  AS proyecto,
       r.descripcion,
       r.observaciones,
       r.km, r.desplazamiento, r.manutencion, r.alojamiento
FROM registros r
JOIN usuarios  u  ON u.id  = r.id_usuario
JOIN proyectos p  ON p.id  = r.id_proyecto
JOIN centros   ce ON ce.id = p.id_centro
JOIN clientes  cl ON cl.id = ce.id_cliente
WHERE cl.nombre ILIKE '%acme%'
  AND translate(lower(p.nombre), 'áéíóúüñÁÉÍÓÚÜÑ', 'aeiouunaeiouun') LIKE '%mantenimiento%'
  AND r.inicio >= DATE '2026-02-01'
  AND r.inicio <  DATE '2026-03-01'
ORDER BY r.inicio
LIMIT 200;
```

**5. Kilómetros y gastos por cliente en 2026.**

```sql
SELECT cl.nombre AS cliente,
       SUM(r.km)             AS km,
       SUM(r.desplazamiento) AS desplazamiento,
       SUM(r.manutencion)    AS manutencion,
       SUM(r.alojamiento)    AS alojamiento,
       COUNT(*) FILTER (WHERE r.km > 0 OR r.desplazamiento > 0 OR r.manutencion > 0 OR r.alojamiento > 0) AS registros_con_gastos
FROM registros r
JOIN proyectos p  ON p.id  = r.id_proyecto
JOIN centros   ce ON ce.id = p.id_centro
JOIN clientes  cl ON cl.id = ce.id_cliente
WHERE r.inicio >= DATE '2026-01-01'
  AND r.inicio <  DATE '2027-01-01'
GROUP BY cl.nombre
HAVING SUM(r.km + r.desplazamiento + r.manutencion + r.alojamiento) > 0
ORDER BY cl.nombre;
```

**6. ¿Qué proyectos activos tiene el cliente Acme?**

```sql
SELECT ce.nombre AS centro,
       p.nombre  AS proyecto,
       p.descripcion,
       to_char(p.inicio, 'DD/MM/YYYY') AS fecha_alta
FROM proyectos p
JOIN centros  ce ON ce.id = p.id_centro
JOIN clientes cl ON cl.id = ce.id_cliente
WHERE cl.nombre ILIKE '%acme%'
  AND p.fin IS NULL
ORDER BY ce.nombre, p.nombre;
```

**7. ¿A qué hora entró y salió Marta cada día de la semana pasada y cuántas horas hizo?**

```sql
SELECT u.nick,
       r.inicio::date AS dia,
       (ARRAY['lunes','martes','miércoles','jueves','viernes','sábado','domingo'])[EXTRACT(ISODOW FROM r.inicio::date)::int] AS dia_semana,
       to_char(MIN(r.inicio), 'HH24:MI') AS entrada,
       to_char(MAX(r.fin),    'HH24:MI') AS salida,
       ROUND(SUM(EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600)::numeric, 2) AS horas
FROM registros r
JOIN usuarios u ON u.id = r.id_usuario
WHERE (u.nick ILIKE '%marta%' OR u.nombre ILIKE '%marta%')
  AND r.inicio >= date_trunc('week', LOCALTIMESTAMP) - INTERVAL '1 week'
  AND r.inicio <  date_trunc('week', LOCALTIMESTAMP)
GROUP BY u.id, u.nick, r.inicio::date
ORDER BY u.nick, dia;
```

**8. ¿Cuál es el último registro de cada usuario?**

```sql
SELECT DISTINCT ON (u.id)
       u.nick,
       p.nombre AS proyecto,
       to_char(r.inicio, 'DD/MM/YYYY HH24:MI') AS desde,
       to_char(r.fin,    'DD/MM/YYYY HH24:MI') AS hasta
FROM usuarios u
JOIN registros r ON r.id_usuario = u.id
JOIN proyectos p ON p.id = r.id_proyecto
ORDER BY u.id, r.inicio DESC;
```

**9. ¿Qué proyectos activos no han tenido actividad en los últimos 90 días?**

```sql
SELECT cl.nombre AS cliente,
       ce.nombre AS centro,
       p.nombre  AS proyecto,
       (SELECT to_char(MAX(r2.inicio), 'DD/MM/YYYY')
        FROM registros r2
        WHERE r2.id_proyecto = p.id) AS ultimo_registro
FROM proyectos p
JOIN centros  ce ON ce.id = p.id_centro
JOIN clientes cl ON cl.id = ce.id_cliente
WHERE p.fin IS NULL
  AND NOT EXISTS (
        SELECT 1
        FROM registros r
        WHERE r.id_proyecto = p.id
          AND r.inicio >= LOCALTIMESTAMP - INTERVAL '90 days')
ORDER BY cl.nombre, ce.nombre, p.nombre;
```

**10. Horas de cada usuario operativo en octubre de 2026, incluidos los que no registraron nada.**

```sql
SELECT u.nick,
       ROUND(COALESCE(SUM(EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600), 0)::numeric, 2) AS horas
FROM usuarios u
LEFT JOIN registros r
       ON r.id_usuario = u.id
      AND r.inicio >= DATE '2026-10-01'
      AND r.inicio <  DATE '2026-11-01'
WHERE u.operativo
GROUP BY u.id, u.nick
ORDER BY horas DESC;
```

**11. Evolución mensual de horas en los últimos 12 meses.**

```sql
SELECT to_char(date_trunc('month', r.inicio), 'YYYY-MM') AS mes,
       COUNT(*) AS registros,
       ROUND(SUM(EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600)::numeric, 2) AS horas
FROM registros r
WHERE r.inicio >= date_trunc('month', LOCALTIMESTAMP) - INTERVAL '11 months'
GROUP BY 1
ORDER BY 1;
```

**12. ¿Qué trabajos mencionan averías?**

```sql
SELECT to_char(r.inicio, 'DD/MM/YYYY') AS fecha,
       u.nick    AS usuario,
       cl.nombre AS cliente,
       p.nombre  AS proyecto,
       r.descripcion,
       r.observaciones
FROM registros r
JOIN usuarios  u  ON u.id  = r.id_usuario
JOIN proyectos p  ON p.id  = r.id_proyecto
JOIN centros   ce ON ce.id = p.id_centro
JOIN clientes  cl ON cl.id = ce.id_cliente
WHERE translate(lower(r.descripcion),   'áéíóúüñÁÉÍÓÚÜÑ', 'aeiouunaeiouun') LIKE '%averia%'
   OR translate(lower(r.observaciones), 'áéíóúüñÁÉÍÓÚÜÑ', 'aeiouunaeiouun') LIKE '%averia%'
ORDER BY r.inicio DESC
LIMIT 100;
```

**13. Los 5 proyectos con más horas este año.**

```sql
SELECT p.nombre  AS proyecto,
       cl.nombre AS cliente,
       ROUND(SUM(EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600)::numeric, 2) AS horas
FROM registros r
JOIN proyectos p  ON p.id  = r.id_proyecto
JOIN centros   ce ON ce.id = p.id_centro
JOIN clientes  cl ON cl.id = ce.id_cliente
WHERE r.inicio >= date_trunc('year', LOCALTIMESTAMP)
  AND r.inicio <  date_trunc('year', LOCALTIMESTAMP) + INTERVAL '1 year'
GROUP BY p.id, p.nombre, cl.nombre
ORDER BY horas DESC
LIMIT 5;
```

**14. Compara las horas por cliente de 2025 y de 2026.**

```sql
SELECT cl.nombre AS cliente,
       ROUND(COALESCE(SUM(EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600)
             FILTER (WHERE r.inicio >= DATE '2025-01-01' AND r.inicio < DATE '2026-01-01'), 0)::numeric, 2) AS horas_2025,
       ROUND(COALESCE(SUM(EXTRACT(EPOCH FROM (COALESCE(r.fin, LOCALTIMESTAMP) - r.inicio)) / 3600)
             FILTER (WHERE r.inicio >= DATE '2026-01-01' AND r.inicio < DATE '2027-01-01'), 0)::numeric, 2) AS horas_2026
FROM registros r
JOIN proyectos p  ON p.id  = r.id_proyecto
JOIN centros   ce ON ce.id = p.id_centro
JOIN clientes  cl ON cl.id = ce.id_cliente
WHERE r.inicio >= DATE '2025-01-01'
  AND r.inicio <  DATE '2027-01-01'
GROUP BY cl.nombre
ORDER BY cl.nombre;
```

**15. Registros sospechosos: más de 12 horas o duración cero.**

```sql
SELECT u.nick   AS usuario,
       p.nombre AS proyecto,
       to_char(r.inicio, 'DD/MM/YYYY HH24:MI') AS desde,
       to_char(r.fin,    'DD/MM/YYYY HH24:MI') AS hasta,
       ROUND((EXTRACT(EPOCH FROM (r.fin - r.inicio)) / 3600)::numeric, 2) AS horas
FROM registros r
JOIN usuarios  u ON u.id = r.id_usuario
JOIN proyectos p ON p.id = r.id_proyecto
WHERE r.fin - r.inicio > INTERVAL '12 hours'
   OR r.fin = r.inicio
ORDER BY horas DESC;
```

**16. ¿Cuántos días con dietas tuvo cada persona en enero de 2026 y cuánto suman?**

```sql
SELECT u.nick AS usuario,
       COUNT(DISTINCT r.inicio::date) AS dias_con_manutencion,
       SUM(r.manutencion)             AS manutencion_total
FROM registros r
JOIN usuarios u ON u.id = r.id_usuario
WHERE r.manutencion > 0
  AND r.inicio >= DATE '2026-01-01'
  AND r.inicio <  DATE '2026-02-01'
GROUP BY u.id, u.nick
ORDER BY u.nick;
```

**17. Lista de usuarios.** (sin `clave`, `email` ni `telefono`)

```sql
SELECT nick, nombre, apellido1, apellido2, administrador, operativo
FROM usuarios
ORDER BY nombre, apellido1;
```

---

## Apéndice: DDL equivalente

```sql
-- Esquema public
CREATE TABLE clientes (
    id      uuid         PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre  varchar(100) NOT NULL UNIQUE,
    abierto boolean      NOT NULL DEFAULT true
);

CREATE TABLE centros (
    id         uuid         PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre     varchar(100) NOT NULL UNIQUE,
    id_cliente uuid         NOT NULL REFERENCES clientes(id) ON UPDATE CASCADE ON DELETE CASCADE,
    abierto    boolean      NOT NULL DEFAULT true
);

CREATE TABLE proyectos (
    id          uuid         PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre      varchar(100) NOT NULL UNIQUE,
    id_centro   uuid         NOT NULL REFERENCES centros(id) ON UPDATE CASCADE ON DELETE CASCADE,
    descripcion text,
    inicio      timestamp    NOT NULL DEFAULT date_trunc('minute', LOCALTIMESTAMP),
    fin         timestamp                      -- NULL = proyecto activo
);

CREATE TABLE usuarios (
    id            uuid         PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre        varchar(100) NOT NULL,
    apellido1     varchar(150) NOT NULL,
    apellido2     varchar(150),
    nick          varchar(50)  NOT NULL UNIQUE,
    clave         text         NOT NULL UNIQUE,  -- ⛔ credencial: nunca consultar
    email         varchar(150) NOT NULL UNIQUE,
    telefono      varchar(20)  NOT NULL UNIQUE,
    administrador boolean      NOT NULL DEFAULT false,
    operativo     boolean      NOT NULL DEFAULT true,
    perfil        varchar(255)                   -- archivo de la foto de perfil
);

CREATE TABLE registros (
    id             uuid          PRIMARY KEY DEFAULT gen_random_uuid(),
    id_proyecto    uuid          NOT NULL REFERENCES proyectos(id) ON UPDATE CASCADE ON DELETE CASCADE,
    inicio         timestamp     NOT NULL DEFAULT date_trunc('minute', LOCALTIMESTAMP),
    fin            timestamp,                    -- NULL = registro en curso
    descripcion    text,
    observaciones  text,
    id_usuario     uuid          NOT NULL REFERENCES usuarios(id) ON UPDATE CASCADE ON DELETE CASCADE,
    km             numeric(10,2) NOT NULL DEFAULT 0,  -- kilómetros
    desplazamiento numeric(10,2) NOT NULL DEFAULT 0,  -- €
    manutencion    numeric(10,2) NOT NULL DEFAULT 0,  -- €
    alojamiento    numeric(10,2) NOT NULL DEFAULT 0   -- €
);

-- Índices: solo los de las PK y las restricciones UNIQUE (no hay índices sobre FK ni sobre fechas).
```
