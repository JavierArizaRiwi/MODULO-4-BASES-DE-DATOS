# Guía Completa de SQL

El lenguaje SQL se divide en varios sublenguajes según la operación que vayamos a realizar. Aquí tienes un resumen de todos ellos:

## DDL (Data Definition Language)
Se usa para **definir y modificar la estructura** de la base de datos (tablas, esquemas, etc.).
- `CREATE`: Crea nuevas tablas o bases de datos.
- `ALTER`: Modifica una tabla existente (ej. agregar una columna).
- `DROP`: Elimina por completo una tabla o base de datos.
- `TRUNCATE`: Vacía todos los datos de una tabla, pero mantiene su estructura.

## DML (Data Manipulation Language)
Se usa para **manipular los datos** almacenados dentro de las tablas.
- `INSERT`: Agrega nuevas filas o registros.
- `UPDATE`: Actualiza datos de registros existentes.
- `DELETE`: Elimina filas específicas según una condición.

## DQL (Data Query Language)
Se usa exclusivamente para **consultar** y obtener información.
- `SELECT`: Obtiene datos de una o más tablas basándose en distintos filtros.

## DCL (Data Control Language)
Controla los **permisos y la seguridad** de la base de datos.
- `GRANT`: Otorga permisos de acceso o edición a un usuario.
- `REVOKE`: Quita permisos que se hayan dado previamente a un usuario.

## TCL (Transaction Control Language)
Administra las **transacciones** (un conjunto de operaciones DML que deben ejecutarse como una sola unidad).
- `BEGIN` / `START TRANSACTION`: Inicia una transacción.
- `COMMIT`: Confirma y guarda los cambios de forma permanente en disco.
- `ROLLBACK`: Deshace los cambios en caso de que ocurra un error antes del `COMMIT`.

---

## Constraints
Son reglas que aseguran que los datos guardados en las tablas sean válidos y consistentes:
- `PRIMARY KEY`: Identificador único e irrepetible para cada fila de la tabla.
- `FOREIGN KEY`: Enlace o referencia a la llave primaria de otra tabla.
- `UNIQUE`: Asegura que no haya valores repetidos en toda esa columna.
- `NOT NULL`: Impide que una columna quede vacía (obliga a insertar un valor).
- `DEFAULT`: Asigna un valor predeterminado si el usuario no proporciona uno.
- `CHECK`: Valida que los datos cumplan una condición específica (ej. `precio > 0`).

## Índices
Los índices funcionan como el índice de un libro; aceleran enormemente la búsqueda de registros en tablas gigantes, pero ocupan espacio extra.
```sql
CREATE INDEX idx_user_email ON users(email);
```

## Vistas
Son consultas que se guardan como si fueran "tablas virtuales", útiles para esconder consultas muy complejas.
```sql
CREATE VIEW user_summary AS SELECT * FROM users;
```

## Funciones
Funciones de agregación integradas que calculan valores a partir de varios registros:
- `COUNT`: Cuenta las filas.
- `SUM`: Suma los valores.
- `AVG`: Calcula el promedio (average).
- `MAX` / `MIN`: Encuentra el valor máximo o mínimo.

## JOINs
Sirven para combinar columnas de una o más tablas en una sola tabla de resultados, basándose en campos comunes.
- `INNER JOIN`: Trae solo las filas que hacen "match" en ambas tablas.
- `LEFT JOIN`: Trae todas las filas de la tabla izquierda, y las que coincidan de la derecha.
- `RIGHT JOIN`: Trae todas las filas de la tabla derecha, y las que coincidan de la izquierda.

## Subconsultas
Son consultas anidadas; es decir, un `SELECT` que se ejecuta dentro de otro `SELECT` (o un `WHERE`).
```sql
SELECT * FROM products
WHERE price > (SELECT AVG(price) FROM products);
```
