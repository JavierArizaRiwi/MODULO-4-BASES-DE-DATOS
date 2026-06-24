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

## 2. Definir entidades y relaciones

Antes de crear tablas, identifica qué cosas existen por separado y cómo se relacionan:

- Un **cliente** puede tener muchos teléfonos y realizar muchas ventas.
- Una **venta** pertenece a un cliente y contiene uno o varios productos.
- Un **producto** puede aparecer en muchas ventas.

La relación entre ventas y productos es muchos a muchos, por lo que requiere una tabla intermedia llamada `sale_items`.

## 3. Aplicar la primera forma normal (1FN)

La 1FN exige que cada columna contenga un único valor: nada de listas como `"Teclado, Mouse"` o `"300111, 301222"`. También cada fila debe poder identificarse de forma única.

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
