# Consultas avanzadas en MySQL — ParqueoÁgil

Guía de consultas avanzadas sobre la base de datos del ParqueoÁgil (control de estacionamiento). Cada consulta incluye su narrativa: qué problema de negocio resuelve, cómo funciona y qué técnica de SQL usa.



## 1. Panorama de tickets abiertos (JOIN múltiple)

 El operador necesita ver, en una sola pantalla, qué vehículos están dentro del parqueadero, en qué zona y espacio están, qué tarifa se les aplica y cuánto tiempo llevan. Esto exige cruzar cinco tablas: `tickets`, `vehiculos`, `espacios`, `zonas` y `tarifas`.

```sql
SELECT
    t.id                                         AS ticket,
    t.placa,
    z.nombre                                     AS zona,
    e.nombre                                     AS espacio,
    ta.nombre                                    AS tarifa,
    t.hora_ingreso,
    TIMESTAMPDIFF(MINUTE, t.hora_ingreso, NOW()) AS minutos_dentro
FROM tickets t
JOIN vehiculos v ON v.id = t.vehiculo_id
JOIN espacios  e ON e.id = t.espacio_id
JOIN zonas     z ON z.id = e.zona_id
JOIN tarifas  ta ON ta.id = t.tarifa_id
WHERE t.estado = 'ABIERTO'
ORDER BY minutos_dentro DESC;
```

 `JOIN` encadenado y `TIMESTAMPDIFF` para calcular duraciones. El filtro por `estado` aprovecha el índice `idx_tickets_estado`.

---

## 2. Vehículos sin cliente y clientes sin vehículos

 `vehiculos.cliente_id` admite `NULL` (vehículos de paso) y un cliente puede no tener vehículos registrados. Para depurar datos conviene listar ambos casos. El patrón clásico es `LEFT JOIN ... WHERE ... IS NULL` (anti-join).

```sql
-- Vehículos de paso (sin cliente asociado)
SELECT v.id, v.placa, v.nombre
FROM vehiculos v
LEFT JOIN clientes c ON c.id = v.cliente_id
WHERE c.id IS NULL;

-- Clientes que aún no registran ningún vehículo
SELECT c.id, c.nombre, c.numero_documento
FROM clientes c
LEFT JOIN vehiculos v ON v.cliente_id = c.id
WHERE v.id IS NULL;
```

 Anti-join: el `LEFT JOIN` conserva todas las filas de la izquierda y el `IS NULL` deja solo las que no tienen pareja.

---

## 3. Espacios libres con NOT EXISTS

La vista `vw_espacios_disponibles` usa `NOT IN`, que funciona pero se comporta mal si la subconsulta devuelve algún `NULL` y suele ser menos eficiente. Esta versión con `NOT EXISTS` es más segura y además muestra el nombre de la zona.

```sql
SELECT e.id, z.nombre AS zona, e.nombre AS espacio
FROM espacios e
JOIN zonas z ON z.id = e.zona_id
WHERE e.is_active = TRUE
  AND z.is_active = TRUE
  AND NOT EXISTS (
      SELECT 1
      FROM tickets t
      WHERE t.espacio_id = e.id
        AND t.estado = 'ABIERTO'
  )
ORDER BY z.nombre, e.nombre;
```

 Subconsulta correlacionada con `EXISTS`/`NOT EXISTS`, preferible a `NOT IN` cuando puede haber `NULL`.

---

## 4. Ocupación por zona

 El supervisor quiere saber cuántos espacios tiene cada zona, cuántos están ocupados y qué porcentaje de ocupación hay en este momento. Se combinan agregaciones condicionales con `COUNT` y un cálculo de porcentaje.

```sql
SELECT
    z.nombre                                              AS zona,
    COUNT(e.id)                                           AS total_espacios,
    COUNT(t.id)                                           AS ocupados,
    COUNT(e.id) - COUNT(t.id)                             AS libres,
    ROUND(100 * COUNT(t.id) / NULLIF(COUNT(e.id), 0), 1)  AS pct_ocupacion
FROM zonas z
LEFT JOIN espacios e ON e.zona_id = z.id AND e.is_active = TRUE
LEFT JOIN tickets  t ON t.espacio_id = e.id AND t.estado = 'ABIERTO'
WHERE z.is_active = TRUE
GROUP BY z.id, z.nombre
ORDER BY pct_ocupacion DESC;
```

 `COUNT(columna)` ignora los `NULL` (por eso cuenta solo los espacios con ticket abierto), y `NULLIF` evita la división por cero.

---

## 5. Cálculo del monto a cobrar

 Al momento de la salida hay que calcular cuánto cobrar según la regla de la tarifa: por hora (se cobra cada hora iniciada), por fracción (cada 30 minutos iniciados) o tarifa plana. `CASE` selecciona la fórmula y `CEIL` redondea hacia arriba porque toda hora o fracción iniciada se cobra completa.

```sql
SELECT
    t.id AS ticket,
    t.placa,
    ta.regla_calculo,
    TIMESTAMPDIFF(MINUTE, t.hora_ingreso, COALESCE(t.hora_salida, NOW())) AS minutos,
    CASE ta.regla_calculo
        WHEN 'POR_HORA' THEN
            CEIL(TIMESTAMPDIFF(MINUTE, t.hora_ingreso, COALESCE(t.hora_salida, NOW())) / 60)
            * ta.valor_base
        WHEN 'POR_FRACCION' THEN
            CEIL(TIMESTAMPDIFF(MINUTE, t.hora_ingreso, COALESCE(t.hora_salida, NOW())) / 30)
            * ta.valor_base
        WHEN 'TARIFA_PLANA' THEN
            ta.valor_base
    END + t.recargo AS total_a_cobrar
FROM tickets t
JOIN tarifas ta ON ta.id = t.tarifa_id
WHERE t.estado = 'ABIERTO';
```

 `CASE` como selector de lógica de negocio, `COALESCE` para usar `NOW()` mientras el ticket sigue abierto y `CEIL` para el redondeo hacia arriba.

---

## 6. Ingresos diarios por método de pago

 Contabilidad quiere ver cuánto entró cada día y cómo se repartió entre efectivo, tarjeta, transferencia y app. Una CTE filtra primero los pagos válidos (completados) y luego un pivote con `SUM(CASE ...)` convierte los métodos en columnas.

```sql
WITH pagos_validos AS (
    SELECT DATE(fecha) AS dia, metodo, monto
    FROM pagos
    WHERE estado = 'COMPLETADO'
)
SELECT
    dia,
    SUM(CASE WHEN metodo = 'EFECTIVO'      THEN monto ELSE 0 END) AS efectivo,
    SUM(CASE WHEN metodo = 'TARJETA'       THEN monto ELSE 0 END) AS tarjeta,
    SUM(CASE WHEN metodo = 'TRANSFERENCIA' THEN monto ELSE 0 END) AS transferencia,
    SUM(CASE WHEN metodo = 'APP'           THEN monto ELSE 0 END) AS app,
    SUM(monto)                                                    AS total_dia
FROM pagos_validos
GROUP BY dia
ORDER BY dia DESC;
```

 CTE con `WITH` para ordenar la lógica y tablas pivote con agregación condicional.

---

## 7. Ranking de clientes por gasto

 Para ofrecer descuentos o abonos a los mejores clientes, se calcula cuánto ha pagado cada uno por tickets de sus vehículos y se genera un ranking. `RANK()` deja empates con el mismo puesto y `DENSE_RANK()` no salta posiciones.

```sql
WITH gasto_cliente AS (
    SELECT
        c.id,
        c.nombre,
        COUNT(DISTINCT t.id) AS visitas,
        SUM(p.monto)         AS total_pagado
    FROM clientes c
    JOIN vehiculos v ON v.cliente_id = c.id
    JOIN tickets   t ON t.vehiculo_id = v.id
    JOIN pagos     p ON p.referencia_tipo = 'TICKET'
                    AND p.referencia_id   = t.id
                    AND p.estado = 'COMPLETADO'
    GROUP BY c.id, c.nombre
)
SELECT
    nombre,
    visitas,
    total_pagado,
    RANK()       OVER (ORDER BY total_pagado DESC) AS puesto,
    DENSE_RANK() OVER (ORDER BY total_pagado DESC) AS puesto_denso,
    ROUND(100 * total_pagado / SUM(total_pagado) OVER (), 2) AS pct_del_total
FROM gasto_cliente
ORDER BY puesto;
```

 Funciones de ventana (`RANK`, `DENSE_RANK`, `SUM() OVER ()`) que calculan sobre el conjunto sin colapsar las filas.

---

## 8. Tiempo entre visitas de una misma placa

 Para detectar clientes frecuentes (candidatos a abono) interesa saber cada cuánto vuelve un vehículo. `LAG()` trae la hora de ingreso de la visita anterior de la misma placa, sin necesidad de un auto-join.

```sql
SELECT
    placa,
    hora_ingreso,
    LAG(hora_ingreso) OVER (PARTITION BY placa ORDER BY hora_ingreso) AS visita_anterior,
    TIMESTAMPDIFF(
        DAY,
        LAG(hora_ingreso) OVER (PARTITION BY placa ORDER BY hora_ingreso),
        hora_ingreso
    ) AS dias_desde_ultima
FROM tickets
ORDER BY placa, hora_ingreso;
```

 `PARTITION BY` reinicia la ventana por cada placa; `LAG` mira la fila anterior dentro de ella.

---

## 9. Acumulado de ingresos del mes

 Para el reporte mensual se quiere ver, día a día, cuánto se facturó y cuánto lleva acumulado en el mes. Se agrupa por día y luego se aplica una suma acumulada con ventana.

```sql
SELECT
    dia,
    total_dia,
    SUM(total_dia) OVER (ORDER BY dia) AS acumulado_mes
FROM (
    SELECT DATE(fecha) AS dia, SUM(monto) AS total_dia
    FROM pagos
    WHERE estado = 'COMPLETADO'
      AND fecha >= DATE_FORMAT(CURDATE(), '%Y-%m-01')
      AND fecha <  DATE_FORMAT(CURDATE(), '%Y-%m-01') + INTERVAL 1 MONTH
    GROUP BY DATE(fecha)
) d
ORDER BY dia;
```

 Suma acumulada (`running total`) con `SUM() OVER (ORDER BY ...)` y filtro de rango de fechas sin usar funciones sobre la columna (`fecha`), lo que permite usar índices.

---

## 10. Afluencia por hora del día
Para decidir cuántos operadores poner en cada franja, se necesita saber a qué horas entran más vehículos. Si una hora no tiene ingresos, igual debe aparecer con cero, así que se genera la serie 0–23 con un **CTE recursivo** y se une con los tickets.

```sql
WITH RECURSIVE horas AS (
    SELECT 0 AS hora
    UNION ALL
    SELECT hora + 1 FROM horas WHERE hora < 23
)
SELECT
    h.hora,
    COUNT(t.id) AS ingresos
FROM horas h
LEFT JOIN tickets t ON HOUR(t.hora_ingreso) = h.hora
GROUP BY h.hora
ORDER BY h.hora;
```

 CTE recursivo para generar series y `LEFT JOIN` para no perder las horas vacías.

---

## 11. Cuadre de caja por turno

Al cerrar un turno, el cajero debe cuadrar: lo que había al abrir más lo cobrado en efectivo debería coincidir con el monto de cierre. La diferencia indica sobrante o faltante. Solo el efectivo entra al cuadre; tarjeta, transferencia y app no pasan por la caja física.

```sql
SELECT
    tc.id                           AS turno,
    u.nombre                        AS cajero,
    tc.fecha_apertura,
    tc.fecha_cierre,
    tc.monto_apertura,
    COALESCE(SUM(p.monto), 0)       AS cobrado_efectivo,
    tc.monto_apertura + COALESCE(SUM(p.monto), 0) AS esperado,
    tc.monto_cierre,
    tc.monto_cierre - (tc.monto_apertura + COALESCE(SUM(p.monto), 0)) AS diferencia
FROM turnos_caja tc
JOIN usuarios u ON u.id = tc.usuario_id
LEFT JOIN tickets t ON t.turno_caja_id = tc.id
LEFT JOIN pagos   p ON p.referencia_tipo = 'TICKET'
                   AND p.referencia_id   = t.id
                   AND p.metodo = 'EFECTIVO'
                   AND p.estado = 'COMPLETADO'
WHERE tc.estado = 'CERRADO'
GROUP BY tc.id, u.nombre, tc.fecha_apertura, tc.fecha_cierre,
         tc.monto_apertura, tc.monto_cierre
HAVING diferencia <> 0
ORDER BY tc.fecha_cierre DESC;
```

 `HAVING` filtra sobre valores agregados (aquí, solo los turnos descuadrados). Ten en cuenta que un mismo ticket con varios pagos en efectivo se suma correctamente porque la agregación ocurre después de los joins.

---

## 12. Pagos unificados de tickets y abonos

 La tabla `pagos` es polimórfica: `referencia_tipo` indica si el pago corresponde a un ticket o a un abono, y `referencia_id` apunta a la tabla correspondiente (sin llave foránea, porque no puede apuntar a dos tablas a la vez). Para un reporte único se resuelve cada caso por separado y se combinan con `UNION ALL`.

```sql
SELECT
    p.id AS pago,
    p.fecha,
    'TICKET'      AS concepto,
    t.placa       AS detalle,
    p.metodo,
    p.monto
FROM pagos p
JOIN tickets t ON p.referencia_tipo = 'TICKET' AND p.referencia_id = t.id
WHERE p.estado = 'COMPLETADO'

UNION ALL

SELECT
    p.id,
    p.fecha,
    'ABONO',
    c.nombre,
    p.metodo,
    p.monto
FROM pagos p
JOIN abonos   a ON p.referencia_tipo = 'ABONO' AND p.referencia_id = a.id
JOIN clientes c ON c.id = a.cliente_id
WHERE p.estado = 'COMPLETADO'

ORDER BY fecha DESC;
```

**Qué aprendes.** Manejo de relaciones polimórficas y `UNION ALL` (más rápido que `UNION` porque no elimina duplicados).

---

## 13. Vehículos con abono activo que además pagan ticket

 Un vehículo con abono vigente no debería generar cobro por ticket. Si aparece pagando, puede haber un error de operación o un abono mal registrado. Se cruza `abono_vehiculo` con `abonos` y se verifica con `EXISTS` que haya tickets con pago.

```sql
SELECT DISTINCT v.placa, c.nombre AS cliente, a.id AS abono
FROM abonos a
JOIN abono_vehiculo av ON av.abono_id = a.id
JOIN vehiculos v       ON v.id = av.vehiculo_id
JOIN clientes c        ON c.id = a.cliente_id
WHERE a.estado = 'ACTIVO'
  AND EXISTS (
      SELECT 1
      FROM tickets t
      JOIN pagos p ON p.referencia_tipo = 'TICKET'
                  AND p.referencia_id   = t.id
                  AND p.estado = 'COMPLETADO'
      WHERE t.vehiculo_id = v.id
        AND t.hora_ingreso >= a.fecha
  );
```

 `EXISTS` con subconsulta correlacionada y tabla puente de relación muchos a muchos.

---

## 14. Tickets con pérdida, recargos y novedades

 Cuando un cliente pierde el ticket se marca `perdida = TRUE` y se aplica un recargo. Para auditoría interesa ver estos casos junto con las novedades registradas por los operadores. `GROUP_CONCAT` junta todas las novedades de un ticket en una sola celda.

```sql
SELECT
    t.id AS ticket,
    t.placa,
    t.hora_ingreso,
    t.recargo,
    COUNT(n.id)                                         AS num_novedades,
    GROUP_CONCAT(n.nombre ORDER BY n.created_at SEPARATOR ' | ') AS novedades
FROM tickets t
LEFT JOIN novedades n ON n.ticket_id = t.id AND n.is_active = TRUE
WHERE t.perdida = TRUE
   OR t.recargo > 0
GROUP BY t.id, t.placa, t.hora_ingreso, t.recargo
ORDER BY t.hora_ingreso DESC;
```

 `GROUP_CONCAT` con `ORDER BY` y `SEPARATOR` para consolidar filas relacionadas.

---

## 15. Detección de inconsistencias

 Una buena base de datos se audita. Estas consultas buscan datos que no deberían existir, aunque las restricciones no los impidan.

```sql
-- a) Más de un ticket ABIERTO para la misma placa
SELECT placa, COUNT(*) AS tickets_abiertos
FROM tickets
WHERE estado = 'ABIERTO'
GROUP BY placa
HAVING COUNT(*) > 1;

-- b) Más de un ticket ABIERTO en el mismo espacio
SELECT espacio_id, COUNT(*) AS tickets_abiertos
FROM tickets
WHERE estado = 'ABIERTO'
GROUP BY espacio_id
HAVING COUNT(*) > 1;

-- c) Tickets cerrados sin hora de salida, o con salida anterior al ingreso
SELECT id, placa, hora_ingreso, hora_salida
FROM tickets
WHERE (estado = 'CERRADO' AND hora_salida IS NULL)
   OR (hora_salida IS NOT NULL AND hora_salida < hora_ingreso);

-- d) La placa guardada en el ticket no coincide con la del vehículo
SELECT t.id, t.placa AS placa_ticket, v.placa AS placa_vehiculo
FROM tickets t
JOIN vehiculos v ON v.id = t.vehiculo_id
WHERE t.placa <> v.placa;

-- e) Pagos de tickets que no existen (referencia huérfana)
SELECT p.id, p.referencia_id
FROM pagos p
LEFT JOIN tickets t ON p.referencia_tipo = 'TICKET' AND t.id = p.referencia_id
WHERE p.referencia_tipo = 'TICKET'
  AND t.id IS NULL;

-- f) Tickets cerrados sin ningún pago completado
SELECT t.id, t.placa, t.monto_calculado
FROM tickets t
LEFT JOIN pagos p ON p.referencia_tipo = 'TICKET'
                 AND p.referencia_id = t.id
                 AND p.estado = 'COMPLETADO'
WHERE t.estado = 'CERRADO'
  AND p.id IS NULL;
```

 Consultas de calidad de datos: `HAVING COUNT(*) > 1` para duplicados y anti-joins para huérfanos. La (e) existe justamente porque `pagos.referencia_id` no tiene llave foránea.

---

## 16. Cierre de ticket seguro con transacción y bloqueo

 Dos cajeros no deben cerrar el mismo ticket a la vez. Se usa una transacción con `SELECT ... FOR UPDATE`, que bloquea la fila hasta el `COMMIT`. Si algo falla, `ROLLBACK` deja todo como estaba. El trigger `trg_tickets_no_doble_cierre` es la última barrera: impide modificar un ticket ya cerrado.

```sql
START TRANSACTION;

-- 1) Bloquear el ticket mientras se procesa
SELECT id, hora_ingreso, tarifa_id, recargo
FROM tickets
WHERE id = 1 AND estado = 'ABIERTO'
FOR UPDATE;

-- 2) Cerrar el ticket calculando el monto (ejemplo con tarifa por hora)
UPDATE tickets t
JOIN tarifas ta ON ta.id = t.tarifa_id
SET t.hora_salida     = NOW(),
    t.estado          = 'CERRADO',
    t.monto_calculado = CEIL(TIMESTAMPDIFF(MINUTE, t.hora_ingreso, NOW()) / 60) * ta.valor_base
                        + t.recargo
WHERE t.id = 1
  AND t.estado = 'ABIERTO';

-- 3) Registrar el pago
INSERT INTO pagos (referencia_tipo, referencia_id, usuario_id, metodo, monto)
SELECT 'TICKET', t.id, 1, 'EFECTIVO', t.monto_calculado
FROM tickets t
WHERE t.id = 1 AND t.estado = 'CERRADO';

COMMIT;
```

 Propiedades ACID en acción, bloqueo pesimista con `FOR UPDATE` y `UPDATE` con `JOIN`. Si el ticket ya estaba cerrado, el segundo `UPDATE` no afecta filas y el `INSERT` no inserta nada.

---

## 17. Procedimiento almacenado para registrar ingresos

 Registrar el ingreso de un vehículo implica varios pasos: buscar o crear el vehículo, verificar que el espacio esté libre y abrir el ticket. Encapsularlo en un procedimiento garantiza que la regla se aplique igual desde cualquier aplicación.

```sql
DELIMITER $$

CREATE PROCEDURE sp_registrar_ingreso(
    IN p_placa      VARCHAR(15),
    IN p_espacio_id INT,
    IN p_tarifa_id  INT,
    IN p_turno_id   INT
)
BEGIN
    DECLARE v_vehiculo_id INT;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;

    START TRANSACTION;

    -- Espacio ocupado: se rechaza el ingreso
    IF EXISTS (SELECT 1 FROM tickets
               WHERE espacio_id = p_espacio_id AND estado = 'ABIERTO') THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'El espacio ya esta ocupado.';
    END IF;

    -- Placa ya dentro del parqueadero: se rechaza el ingreso
    IF EXISTS (SELECT 1 FROM tickets
               WHERE placa = p_placa AND estado = 'ABIERTO') THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'La placa ya tiene un ticket abierto.';
    END IF;

    -- Buscar el vehículo; si no existe, crearlo
    SELECT id INTO v_vehiculo_id FROM vehiculos WHERE placa = p_placa;

    IF v_vehiculo_id IS NULL THEN
        INSERT INTO vehiculos (placa) VALUES (p_placa);
        SET v_vehiculo_id = LAST_INSERT_ID();
    END IF;

    INSERT INTO tickets (vehiculo_id, espacio_id, tarifa_id, turno_caja_id, placa, hora_ingreso)
    VALUES (v_vehiculo_id, p_espacio_id, p_tarifa_id, p_turno_id, p_placa, NOW());

    COMMIT;
END$$

DELIMITER ;

-- Uso
CALL sp_registrar_ingreso('ABC123', 1, 1, 1);
```

 Procedimientos con parámetros, manejo de errores (`EXIT HANDLER`, `SIGNAL`, `RESIGNAL`) y transacciones dentro del procedimiento. Nota: el turno de caja con id 1 debe existir antes de llamarlo (la carga inicial no lo crea).

---

## 18. Optimización con EXPLAIN e índices

 Con miles de tickets, algunas consultas se vuelven lentas. `EXPLAIN ANALYZE` muestra cómo ejecuta MySQL la consulta y cuánto tarda cada paso. Estos son los índices que más ayudan según las consultas de esta guía.

```sql
-- Ver el plan de ejecución real
EXPLAIN ANALYZE
SELECT placa, COUNT(*) AS visitas
FROM tickets
WHERE hora_ingreso >= '2026-01-01'
GROUP BY placa;

-- Índices recomendados
CREATE INDEX idx_tickets_hora_ingreso  ON tickets (hora_ingreso);
CREATE INDEX idx_tickets_espacio_estado ON tickets (espacio_id, estado);
CREATE INDEX idx_tickets_vehiculo       ON tickets (vehiculo_id);
CREATE INDEX idx_tickets_turno          ON tickets (turno_caja_id);
CREATE INDEX idx_pagos_fecha_estado     ON pagos (fecha, estado);
```


| Señal en EXPLAIN | Significado |
|---|---|
| `type: ALL` | Recorre toda la tabla (revisar índices). |
| `type: ref` / `range` | Usa un índice, buen resultado. |
| `Using filesort` | Ordena en memoria o disco; un índice adecuado puede evitarlo. |
| `Using temporary` | Crea una tabla temporal, frecuente en `GROUP BY` sin índice. |

 Un índice compuesto como `(espacio_id, estado)` sirve exactamente al patrón `NOT EXISTS` de la consulta 3, y `idx_tickets_hora_ingreso` acelera las consultas 8, 9 y 10. Recuerda que cada índice agrega costo a las escrituras, así que conviene crear solo los que se usan.

---
