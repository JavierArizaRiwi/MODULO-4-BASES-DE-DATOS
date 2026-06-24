# Consultas SQL avanzadas con PostgreSQL

Estos ejemplos usan las tablas `students`, `courses` y `enrollments` del README. Puedes ejecutarlos después de cargar los datos de prueba.

## 1. Agrupar, filtrar grupos y ordenar

`WHERE` filtra filas antes de agrupar; `HAVING` filtra los grupos que ya fueron calculados.

```sql
SELECT
    c.name AS course_name,
    COUNT(e.student_id) AS total_students
FROM courses c
LEFT JOIN enrollments e ON e.course_id = c.id
GROUP BY c.id, c.name
HAVING COUNT(e.student_id) >= 1
ORDER BY total_students DESC, course_name;
```

## 2. Subconsultas: comparar con un promedio

Encuentra los cursos con más estudiantes que el promedio de inscripciones por curso.

```sql
SELECT c.name, COUNT(e.student_id) AS total_students
FROM courses c
LEFT JOIN enrollments e ON e.course_id = c.id
GROUP BY c.id, c.name
HAVING COUNT(e.student_id) > (
    SELECT AVG(total_students)
    FROM (
        SELECT COUNT(*) AS total_students
        FROM enrollments
        GROUP BY course_id
    ) AS enrollment_counts
);
```

## 3. `EXISTS` y `NOT EXISTS`

`EXISTS` es útil cuando importa saber si hay una relación, no traer sus columnas. Este ejemplo obtiene estudiantes inscritos en al menos un curso y luego los que no tienen inscripciones.

```sql
SELECT s.id, s.first_name, s.last_name
FROM students s
WHERE EXISTS (
    SELECT 1
    FROM enrollments e
    WHERE e.student_id = s.id
);

SELECT s.id, s.first_name, s.last_name
FROM students s
WHERE NOT EXISTS (
    SELECT 1
    FROM enrollments e
    WHERE e.student_id = s.id
);
```

## 4. CTE: consultas legibles con `WITH`

Una CTE asigna nombre a un resultado temporal dentro de una sola consulta. Aquí se cuentan las inscripciones y se clasifican los cursos:

```sql
WITH enrollment_totals AS (
    SELECT course_id, COUNT(*) AS total_students
    FROM enrollments
    GROUP BY course_id
)
SELECT
    c.name,
    COALESCE(et.total_students, 0) AS total_students,
    CASE
        WHEN COALESCE(et.total_students, 0) >= 10 THEN 'alta demanda'
        WHEN COALESCE(et.total_students, 0) >= 5 THEN 'demanda media'
        ELSE 'demanda inicial'
    END AS demand_level
FROM courses c
LEFT JOIN enrollment_totals et ON et.course_id = c.id
ORDER BY total_students DESC;
```

`COALESCE(valor, alternativa)` reemplaza los valores `NULL`, por ejemplo un curso sin inscripciones.

## 5. Funciones de ventana: ranking sin perder filas

Las funciones de ventana calculan sobre un conjunto relacionado de filas sin reducirlas a una sola fila como lo hace `GROUP BY`.

```sql
WITH course_totals AS (
    SELECT c.id, c.name, COUNT(e.student_id) AS total_students
    FROM courses c
    LEFT JOIN enrollments e ON e.course_id = c.id
    GROUP BY c.id, c.name
)
SELECT
    name,
    total_students,
    RANK() OVER (ORDER BY total_students DESC) AS ranking,
    ROUND(100.0 * total_students / SUM(total_students) OVER (), 2) AS percentage_of_enrollments
FROM course_totals
ORDER BY ranking, name;
```

Funciones especialmente útiles: `ROW_NUMBER()` (posición única), `RANK()` (comparte puesto si hay empate) y `LAG()`/`LEAD()` (ver fila anterior o siguiente).

## 6. Acumulados y comparación con la fila anterior

Si se registran varias inscripciones en fechas distintas, se puede calcular un acumulado por curso:

```sql
SELECT
    course_id,
    enrollment_date,
    COUNT(*) AS enrollments_that_day,
    SUM(COUNT(*)) OVER (
        PARTITION BY course_id
        ORDER BY enrollment_date
    ) AS running_total
FROM enrollments
GROUP BY course_id, enrollment_date
ORDER BY course_id, enrollment_date;
```

## 7. CTE recursiva: recorrer una jerarquía

Las CTE recursivas sirven para categorías, equipos o comentarios con relación padre-hijo. Ejemplo con una tabla `categories(id, name, parent_id)`:

```sql
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 AS level, name::TEXT AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    SELECT c.id, c.name, c.parent_id, ct.level + 1, ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT *
FROM category_tree
ORDER BY path;
```

## 8. Operaciones de conjuntos

Usa `UNION` para combinar resultados y eliminar duplicados; `UNION ALL` conserva duplicados y suele ser más rápido. `INTERSECT` devuelve coincidencias y `EXCEPT` devuelve filas presentes en la primera consulta pero no en la segunda.

```sql
-- Estudiantes inscritos en el curso 1 pero no en el curso 2
SELECT student_id FROM enrollments WHERE course_id = 1
EXCEPT
SELECT student_id FROM enrollments WHERE course_id = 2;
```

## 9. Modificar y devolver datos con `RETURNING`

PostgreSQL puede devolver las filas afectadas inmediatamente. Es muy útil para obtener IDs nuevos sin ejecutar otro `SELECT`.

```sql
INSERT INTO students (first_name, last_name, email)
VALUES ('María', 'Gómez', 'maria@example.com')
RETURNING id, first_name, email;

UPDATE courses
SET description = 'Curso actualizado de SQL y modelado de datos'
WHERE id = 1
RETURNING id, name, description;
```

## 10. Analizar rendimiento con `EXPLAIN ANALYZE`

Antes de añadir índices, mide. `EXPLAIN ANALYZE` ejecuta la consulta y muestra el plan real, por lo que no debe usarse con consultas que modifiquen datos.

```sql
EXPLAIN ANALYZE
SELECT s.first_name, s.last_name, c.name
FROM enrollments e
JOIN students s ON s.id = e.student_id
JOIN courses c ON c.id = e.course_id
WHERE e.course_id = 1;

CREATE INDEX idx_enrollments_course_id ON enrollments(course_id);
```

Un índice ayuda cuando la tabla crece y se filtra o une frecuentemente por esa columna. No agregues índices por costumbre: también tienen costo en espacio y escrituras.

## Reto práctico

Escribe una consulta que muestre, por cada estudiante, la cantidad de cursos en los que está inscrito; incluye también a quienes tienen cero cursos. Después añade un ranking por cantidad de cursos usando `RANK() OVER (...)`.
