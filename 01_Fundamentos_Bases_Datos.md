# Fundamentos de Bases de Datos

Bienvenido a la introducción teórica de Bases de Datos. Aquí aprenderás los conceptos fundamentales necesarios antes de empezar a escribir código SQL.

## ¿Qué es una Base de Datos?
Una base de datos (BD) es un sistema estructurado diseñado para almacenar, organizar y recuperar información de manera rápida y segura. Imagínala como un archivador digital súper eficiente donde la información está clasificada en tablas, filas y columnas para que no se pierda y sea fácil de consultar.

## Motores de Bases de Datos Relacionales (SQL)
Son los programas encargados de procesar, almacenar y administrar los datos. Algunos de los más populares en la industria son:
- **PostgreSQL:** Muy potente, open-source y el que usaremos en la práctica.
- **MySQL / MariaDB:** Populares en desarrollo web tradicional.
- **Oracle & SQL Server:** Muy utilizados en entornos corporativos y empresariales.

## SGBD (Sistema de Gestión de Bases de Datos)
Es el software o interfaz (como DBeaver, pgAdmin o DataGrip) que interactúa con el Motor de Base de Datos. Nos proporciona herramientas visuales o consolas que nos permiten:
- Crear y modificar tablas visualmente.
- Gestionar usuarios y administrar permisos.
- Ejecutar consultas (Queries) para extraer información.

---

## Cardinalidades (Relaciones entre tablas)
En una base de datos relacional, los datos se conectan entre sí. La cardinalidad define cómo se relacionan las filas de una tabla con las de otra.

### 1:1 (Uno a Uno)
Un registro en la Tabla A se relaciona con un solo registro en la Tabla B.
* **Ejemplo:** `Usuario -> Perfil` (Un usuario tiene un único perfil de cuenta, y ese perfil pertenece solo a ese usuario).

### 1:N (Uno a Muchos)
Un registro en la Tabla A se relaciona con varios registros en la Tabla B, pero un registro de B solo pertenece a un A.
* **Ejemplo:** `Cliente -> Pedidos` (Un cliente puede hacer muchos pedidos, pero cada pedido específico pertenece a un solo cliente).

### N:M (Muchos a Muchos)
Varios registros de la Tabla A se relacionan con varios registros de la Tabla B. *Nota: En bases de datos esto requiere una tercera tabla "intermedia".*
* **Ejemplo:** `Estudiantes -> Cursos` (Un estudiante se inscribe en múltiples cursos, y un curso tiene matriculados a muchos estudiantes).

---

## Normalización
Es el proceso de diseñar y estructurar las tablas para evitar **redundancia** (datos repetidos) y asegurar la **integridad**.

### 1FN (Primera Forma Normal)
**Regla:** Valores atómicos. No se permiten listas ni grupos de datos en una sola celda (Ej: No puedes tener un solo campo "telefonos" con "300123, 312456").

### 2FN (Segunda Forma Normal)
**Regla:** Eliminar dependencias parciales. Toda columna que no sea clave debe depender completamente de toda la clave principal, no solo de una parte de ella.

### 3FN (Tercera Forma Normal)
**Regla:** Eliminar dependencias transitivas. Las columnas que no son clave no deben depender de otros campos que tampoco son clave (Ej: Si tienes una tabla Empleados, la columna "Nombre_Ciudad" no debe estar ahí si ya tienes un "Ciudad_ID").
