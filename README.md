# 🧾 Actividad: Uso de EVENTOS en MySQL

A continuación se detallan distintos ejemplos del uso de **EVENTOS** en MySQL utilizando las tablas `resumen_ventas` y `alerta_stock`, aplicando tanto `ON COMPLETION PRESERVE` como `ON COMPLETION NOT PRESERVE`.

---

## 📊 Tablas Utilizadas

```sql
CREATE TABLE IF NOT EXISTS resumen_ventas (
    fecha DATE PRIMARY KEY,
    total_pedidos INT,
    total_ingresos DECIMAL(12,2),
    creado_en DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS alerta_stock (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ingrediente_id INT UNSIGNED NOT NULL,
    stock_actual INT NOT NULL,
    fecha_alerta DATETIME NOT NULL,
    creado_en DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (ingrediente_id) REFERENCES ingrediente(id)
);
```

---

## 1️⃣ Resumen Diario Único

**🎯 Objetivo:**  
Crear un evento que genere un resumen de ventas una sola vez al finalizar el día anterior y luego se elimine automáticamente.

```sql
DELIMITER //

CREATE PROCEDURE GenerarResumenDiario()
BEGIN
    INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos)
    SELECT 
        CURDATE() - INTERVAL 1 DAY,
        COUNT(*) AS total_pedidos,
        SUM(monto_pedido) AS total_ingresos
    FROM pedidos
    WHERE DATE(fecha_pedido) = CURDATE() - INTERVAL 1 DAY;
END //

DELIMITER ;

DROP EVENT IF EXISTS generar_resumen_diario_una_vez;

DELIMITER //

CREATE EVENT generar_resumen_diario_una_vez
ON SCHEDULE AT TIMESTAMP '2025-07-10 00:00:00'
ON COMPLETION NOT PRESERVE
DO
    CALL GenerarResumenDiario();
//

DELIMITER ;
```

---

## 2️⃣ Resumen Semanal Recurrente

**🎯 Objetivo:**  
Generar un resumen de pedidos semanal cada lunes a la 01:00 AM, manteniendo el evento activo de forma indefinida.

```sql
CREATE TABLE IF NOT EXISTS resumen_semanal (
    semana_inicio DATE PRIMARY KEY,
    total_pedidos INT,
    total_ingresos DECIMAL(12,2),
    creado_en DATETIME DEFAULT CURRENT_TIMESTAMP
);

DELIMITER //

CREATE PROCEDURE ResumenSemanal()
BEGIN
    INSERT INTO resumen_semanal (semana_inicio, total_pedidos, total_ingresos)
    SELECT 
        CURDATE() - INTERVAL 1 WEEK AS semana_inicio,
        COUNT(*) AS total_pedidos,
        SUM(monto_pedido) AS total_ingresos
    FROM pedidos
    WHERE fecha_pedido >= CURDATE() - INTERVAL 1 WEEK
      AND fecha_pedido < CURDATE();
END //

DELIMITER ;

DROP EVENT IF EXISTS ev_resumen_semanal;

DELIMITER //

CREATE EVENT ev_resumen_semanal
ON SCHEDULE
    EVERY 1 WEEK
    STARTS TIMESTAMP '2025-07-14 01:00:00'
ON COMPLETION PRESERVE
DO
    CALL ResumenSemanal();
//

DELIMITER ;
```

---

## 3️⃣ Alerta de Stock Bajo Única

**🎯 Objetivo:**  
Ejecutar una sola vez para insertar alertas en la tabla `alerta_stock` si hay ingredientes con `stock < 5`. El evento se elimina automáticamente tras su ejecución.

```sql
DELIMITER //

CREATE PROCEDURE ComprobarStock()
BEGIN
    INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta)
    SELECT id, stock, NOW()
    FROM ingrediente
    WHERE stock < 5;
END //

DELIMITER ;

DROP EVENT IF EXISTS ev_stock_bajo;

DELIMITER //

CREATE EVENT ev_stock_bajo
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 1 MINUTE
ON COMPLETION NOT PRESERVE
DO
    CALL ComprobarStock();
//

DELIMITER ;
```

---

## 4️⃣ Monitoreo Continuo de Stock

**🎯 Objetivo:**  
Revisar cada 30 minutos los ingredientes con `stock < 10` e insertar alertas. Este evento se mantiene activo de forma indefinida.

```sql
DELIMITER //

CREATE PROCEDURE monitoreo_stock()
BEGIN
    INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta)
    SELECT id, stock, NOW()
    FROM ingrediente
    WHERE stock < 10;
END //

DELIMITER ;

DROP EVENT IF EXISTS evento_stock_bajo;

DELIMITER //

CREATE EVENT evento_stock_bajo
ON SCHEDULE 
    EVERY 30 MINUTE
    STARTS CURRENT_TIMESTAMP
ON COMPLETION PRESERVE
DO
    CALL monitoreo_stock();
//

DELIMITER ;
```
