# Normalización de bases de datos: guía paso a paso

La normalización organiza los datos para evitar repeticiones innecesarias y errores al insertar, actualizar o eliminar información. En esta guía convertiremos una tabla de ventas desordenada en un diseño en **tercera forma normal (3FN)**.

## 1. Detectar el problema

Imagina que se recibe la siguiente información en una sola tabla. Es cómoda para una hoja de cálculo, pero problemática para una base de datos:

| venta_id | fecha | cliente_nombre | cliente_email | cliente_telefonos | productos |
| --- | --- | --- | --- | --- | --- |
| 1 | 2026-06-01 | Ana López | ana@email.com | 300111, 301222 | Teclado, Mouse |
| 2 | 2026-06-02 | Ana López | ana@email.com | 300111, 301222 | Monitor |

Aquí aparecen tres problemas clásicos:

- **Actualización:** si Ana cambia de correo, hay que modificar todas sus ventas.
- **Inserción:** no es posible registrar un producto si todavía no se ha vendido.
- **Eliminación:** al borrar la última venta de Ana podríamos perder sus datos de contacto.

### Ejemplo de cada anomalía

| Situación | Qué ocurre en la tabla original | Consecuencia |
| --- | --- | --- |
| Ana cambia su correo | Hay que cambiar `ana@email.com` en todas sus ventas. | Si una fila queda sin actualizar, habrá dos correos para la misma persona. |
| Se crea el producto "Cámara" | La tabla exige un `venta_id` para guardar el producto. | No se puede registrar el catálogo hasta vender el producto. |
| Se elimina la venta 2 de Ana y era su última venta | Se borra la única fila con su nombre, correo y teléfonos. | Se pierde información del cliente aunque solo queríamos eliminar una venta. |

## 2. Definir entidades y relaciones

Antes de crear tablas, identifica qué cosas existen por separado y cómo se relacionan:

- Un **cliente** puede tener muchos teléfonos y realizar muchas ventas.
- Una **venta** pertenece a un cliente y contiene uno o varios productos.
- Un **producto** puede aparecer en muchas ventas.

La relación entre ventas y productos es muchos a muchos, por lo que requiere una tabla intermedia llamada `sale_items`.

### Cómo reconocer una relación muchos a muchos

Pregunta ambas direcciones:

- ¿Una venta puede tener varios productos? Sí: por ejemplo, la venta 1 contiene teclado y mouse.
- ¿Un producto puede estar en varias ventas? Sí: el mismo mouse puede venderse a Ana hoy y a Carlos mañana.

Como ambas respuestas son sí, no se debe guardar una lista de productos dentro de `sales` ni una lista de ventas dentro de `products`. La tabla `sale_items` guarda una fila por cada pareja venta-producto:

| sale_id | product_id | quantity |
| --- | --- | --- |
| 1 | 10 (Teclado) | 1 |
| 1 | 11 (Mouse) | 2 |
| 2 | 11 (Mouse) | 1 |

## 3. Aplicar la primera forma normal (1FN)

La 1FN exige que cada columna contenga un único valor: nada de listas como `"Teclado, Mouse"` o `"300111, 301222"`. También cada fila debe poder identificarse de forma única.

### Ejemplo 1: lista de teléfonos

Esto **no** cumple 1FN porque la columna contiene dos valores:

| customer_id | full_name | phones |
| --- | --- | --- |
| 1 | Ana López | 300111, 301222 |

Una corrección es usar una tabla para los teléfonos:

| customer_id | full_name |
| --- | --- |
| 1 | Ana López |

| id | customer_id | phone |
| --- | --- | --- |
| 1 | 1 | 300111 |
| 2 | 1 | 301222 |

Ahora se puede buscar, validar o eliminar un teléfono individual sin manipular texto separado por comas.

### Ejemplo 2: columnas repetidas

También incumple 1FN crear columnas como `product_1`, `product_2` y `product_3`:

| sale_id | product_1 | product_2 | product_3 |
| --- | --- | --- | --- |
| 1 | Teclado | Mouse | NULL |

¿Qué ocurre cuando una venta tiene cuatro productos? La estructura deja de servir. La solución no es añadir `product_4`; es crear filas en `sale_items`.

Separamos los productos de una venta en filas individuales y los teléfonos en su propia tabla. Todavía habrá datos repetidos, pero ya son valores atómicos.

```sql
CREATE TABLE sales_1fn (
    sale_id INTEGER,
    sale_date DATE NOT NULL,
    customer_name VARCHAR(120) NOT NULL,
    customer_email VARCHAR(150) NOT NULL,
    product_name VARCHAR(120) NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (sale_id, product_name)
);
```

La clave compuesta `(sale_id, product_name)` identifica cada producto dentro de una venta. Sin embargo, `sale_date` depende solo de `sale_id`, y los datos del cliente tampoco dependen del producto. Eso incumple 2FN.

## 4. Aplicar la segunda forma normal (2FN)

La 2FN parte de 1FN y elimina dependencias parciales: todo atributo que no sea clave debe depender de **toda** la clave primaria, no de una parte.

Una **dependencia funcional** se lee como “un dato determina otro”. Por ejemplo, `student_id → student_name` significa que, dado un identificador de estudiante, existe un único nombre asociado. La 2FN es relevante cuando la clave primaria está formada por dos o más columnas.

### Ejemplo 1: nota de una inscripción

Supón esta tabla, cuya clave primaria es `(student_id, course_id)`:

| student_id | course_id | student_name | course_name | final_grade |
| --- | --- | --- | --- | --- |
| 1 | 10 | Ana López | SQL | 4.5 |
| 1 | 20 | Ana López | Java | 4.0 |

Las dependencias son:

- `(student_id, course_id) → final_grade`: la nota corresponde a esa inscripción específica. **Correcto**.
- `student_id → student_name`: el nombre depende solo de parte de la clave. **Incorrecto para 2FN**.
- `course_id → course_name`: el nombre del curso depende solo de la otra parte. **Incorrecto para 2FN**.

La corrección separa las entidades:

```sql
CREATE TABLE students (
    id INTEGER PRIMARY KEY,
    full_name VARCHAR(120) NOT NULL
);

CREATE TABLE courses (
    id INTEGER PRIMARY KEY,
    name VARCHAR(120) NOT NULL
);

CREATE TABLE enrollments (
    student_id INTEGER REFERENCES students(id),
    course_id INTEGER REFERENCES courses(id),
    final_grade NUMERIC(3, 1),
    PRIMARY KEY (student_id, course_id)
);
```

El nombre de Ana se actualiza una sola vez en `students`, aunque se inscriba en cien cursos.

### Ejemplo 2: detalle de pedido

En una tabla con clave `(order_id, product_id)`, `quantity` depende de los dos valores: el pedido 50 puede tener 1 mouse y 3 teclados. Por eso pertenece al detalle. En cambio, `order_date` depende solo de `order_id` y `product_name` depende solo de `product_id`; ambos deben salir de la tabla de detalle.

| Atributo | Depende de | Lugar correcto |
| --- | --- | --- |
| `quantity` | `order_id` + `product_id` | `order_items` |
| `order_date` | `order_id` | `orders` |
| `product_name` | `product_id` | `products` |

En `sales_1fn`, `sale_date`, `customer_name` y `customer_email` dependen de `sale_id`, mientras que `quantity` sí depende de la combinación venta-producto. Separamos la venta de sus líneas:

```sql
CREATE TABLE sales_2fn (
    id SERIAL PRIMARY KEY,
    sale_date DATE NOT NULL,
    customer_name VARCHAR(120) NOT NULL,
    customer_email VARCHAR(150) NOT NULL
);

CREATE TABLE sale_items_2fn (
    sale_id INTEGER NOT NULL REFERENCES sales_2fn(id),
    product_name VARCHAR(120) NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(12, 2) NOT NULL CHECK (unit_price >= 0),
    PRIMARY KEY (sale_id, product_name)
);
```

Ahora una venta no repite su fecha por cada producto. Pero el nombre y correo del cliente siguen repitiéndose en cada venta, y el precio actual o la información del producto quedaría repetida en muchas líneas. Esas son dependencias transitivas.

## 5. Aplicar la tercera forma normal (3FN)

La 3FN elimina dependencias transitivas: un dato que no es clave no debe depender de otro dato que tampoco es clave. En este caso:

- `customer_name` depende de `customer_email`/cliente, no de la venta.
- Los datos de un producto dependen del producto, no de una línea de venta.
- Los teléfonos dependen del cliente y puede haber varios.

### Ejemplo 1: ciudad y departamento

Observa una tabla de empleados:

| employee_id | employee_name | city_id | city_name | department |
| --- | --- | --- | --- | --- |
| 1 | Luis | 5 | Medellín | Antioquia |

La clave es `employee_id`. Sin embargo, `city_name` y `department` dependen de `city_id`, no directamente del empleado:

`employee_id → city_id → city_name, department`

Es una dependencia transitiva. La solución es crear `cities(id, name, department)` y conservar solo `city_id` en `employees`. Así, si cambia el nombre de una ciudad, se actualiza una sola fila.

### Ejemplo 2: datos de cliente dentro de una venta

En `sales(id, customer_id, customer_email, sale_date)`, el correo depende de `customer_id`, no de `sales.id`:

`sale_id → customer_id → customer_email`

Por tanto, `customer_email` debe estar en `customers`, y `sales` solo mantiene `customer_id` como llave foránea. No se repite el correo en cada venta.

El diseño final queda así:

```sql
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    full_name VARCHAR(120) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE
);

CREATE TABLE customer_phones (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    phone VARCHAR(30) NOT NULL,
    UNIQUE (customer_id, phone)
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(120) NOT NULL UNIQUE,
    current_price NUMERIC(12, 2) NOT NULL CHECK (current_price >= 0)
);

CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    sale_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE sale_items (
    sale_id INTEGER NOT NULL REFERENCES sales(id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(12, 2) NOT NULL CHECK (unit_price >= 0),
    PRIMARY KEY (sale_id, product_id)
);
```

`unit_price` se conserva en `sale_items` a propósito: representa el precio histórico cobrado en esa venta. `current_price` representa el precio vigente del catálogo; son hechos distintos, no duplicación accidental.

### Caso que parece incumplir 3FN, pero no lo hace

Si hoy un teclado cuesta $85.000 y el próximo mes cuesta $95.000, una venta anterior debe seguir mostrando $85.000. Por eso estas dos columnas tienen sentidos diferentes:

| Tabla | Columna | Significado |
| --- | --- | --- |
| `products` | `current_price` | Precio de catálogo que aplica ahora. |
| `sale_items` | `unit_price` | Precio acordado en una venta concreta. |

Normalizar no significa eliminar toda repetición textual; significa evitar guardar dos veces el **mismo hecho** sin necesidad.

## 6. Comprobar el resultado

Para registrar una venta, primero existen el cliente y los productos; luego se crea la venta y sus líneas. Hazlo dentro de una transacción para que todo se guarde o nada cambie:

```sql
BEGIN;

INSERT INTO customers (full_name, email)
VALUES ('Ana López', 'ana@email.com')
ON CONFLICT (email) DO NOTHING;

INSERT INTO sales (customer_id)
SELECT id FROM customers WHERE email = 'ana@email.com';

INSERT INTO sale_items (sale_id, product_id, quantity, unit_price)
VALUES
    (1, 1, 1, 85000.00),
    (1, 2, 2, 30000.00);

COMMIT;
```

> En una aplicación real, obtén el identificador de la venta con `INSERT ... RETURNING id` en vez de asumir que es `1`.

## 7. Lista de verificación antes de terminar

- ¿Cada celda contiene un solo valor? Si no, falta 1FN.
- Si hay una clave compuesta, ¿cada columna descriptiva depende de la clave completa? Si no, falta 2FN.
- ¿Un atributo depende de otra columna que no es clave? Muévelo a la entidad que representa ese dato: falta 3FN.
- ¿Las relaciones muchos a muchos tienen una tabla intermedia?
- ¿Las reglas importantes están protegidas con `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `NOT NULL` y `CHECK`?

Normalizar no consiste en crear la mayor cantidad de tablas posible; consiste en que cada tabla represente un hecho claro y que la base de datos pueda proteger sus relaciones.

## Resumen rápido con un mismo ejemplo

| Forma normal | Pregunta que debes hacer | Problema típico | Solución |
| --- | --- | --- | --- |
| 1FN | ¿Hay listas, grupos o columnas repetidas? | `phones = '300111, 301222'` | Una fila por teléfono en `customer_phones`. |
| 2FN | Si la clave es compuesta, ¿cada dato depende de toda ella? | `student_name` dentro de `(student_id, course_id)` | Mover el nombre a `students`. |
| 3FN | ¿Un dato no clave depende de otro dato no clave? | `city_name` depende de `city_id` dentro de `employees` | Crear `cities` y referenciarla con `city_id`. |
