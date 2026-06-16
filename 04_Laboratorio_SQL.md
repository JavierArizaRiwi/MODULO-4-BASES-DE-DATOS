# Laboratorio SQL

En este laboratorio crearemos un sistema básico de e-commerce simulando tres entidades: **Usuarios, Productos y Pedidos (Órdenes)**.

## 1. Tabla Users (Usuarios)

Usaremos `SERIAL` para que la base de datos nos genere el ID automáticamente cada vez que agreguemos alguien. Además, impediremos que existan dos correos iguales con `UNIQUE`.
```sql
CREATE TABLE users(
 id SERIAL PRIMARY KEY,
 full_name VARCHAR(100),
 email VARCHAR(150) UNIQUE
);
```

## Tabla Products

```sql
CREATE TABLE products(
 id SERIAL PRIMARY KEY,
 name VARCHAR(100),
 price NUMERIC(10,2)
);
```

## Tabla Orders

```sql
CREATE TABLE orders(
 id SERIAL PRIMARY KEY,
 user_id INTEGER REFERENCES users(id),
 product_id INTEGER REFERENCES products(id),
 quantity INTEGER
);
```

## JOIN

```sql
SELECT u.full_name,p.name,o.quantity
FROM orders o
INNER JOIN users u ON u.id=o.user_id
INNER JOIN products p ON p.id=o.product_id;
```
