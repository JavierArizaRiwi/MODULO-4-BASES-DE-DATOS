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
Es el proceso de diseñar y estructurar las tablas de una base de datos para minimizar la **redundancia** (datos repetidos) y mejorar la **integridad** de los datos. Seguir las formas normales ayuda a prevenir problemas de actualización y a mantener una base de datos lógica y organizada.

### 1FN (Primera Forma Normal)
**Regla: Atomicidad de los datos.** Cada celda de una tabla debe contener un único valor, y no pueden existir grupos o listas de valores en una misma celda.
*   **❌ Incorrecto:** Una columna `telefonos` con el valor `"300123, 312456"`.
*   **✅ Correcto:** Crear una tabla separada `telefonos_usuario` que relacione al usuario con sus múltiples números.

Otro caso incorrecto es tener columnas `curso_1`, `curso_2` y `curso_3`. Si un estudiante toma un cuarto curso, habría que cambiar la estructura. La solución es una tabla de inscripciones: una fila por cada estudiante y curso.

### 2FN (Segunda Forma Normal)
**Regla: Dependencia funcional completa.** Se aplica a tablas con claves primarias compuestas (más de una columna). Establece que todos los atributos que no son clave deben depender de la clave primaria completa, no solo de una parte de ella.

**Ejemplo:** en `inscripciones(student_id, course_id, student_name, course_name, grade)`, la clave es `(student_id, course_id)`. La columna `grade` sí depende de la inscripción completa; pero `student_name` depende solo de `student_id` y `course_name` solo de `course_id`. Para cumplir 2FN, se crean las tablas `students`, `courses` e `enrollments`, dejando en `enrollments` solo los datos propios de la relación, como la nota o la fecha de inscripción.

### 3FN (Tercera Forma Normal)
**Regla: Eliminar dependencias transitivas.** Ningún atributo que no sea clave debe depender de otro atributo que tampoco es clave.
*   **Ejemplo:** Si tienes una tabla `pedidos` con columnas `cliente_id`, `cliente_direccion` y `fecha_pedido`. La dirección del cliente (`cliente_direccion`) depende del cliente (`cliente_id`), no del pedido en sí.
*   **✅ Correcto:** La dirección debe estar en la tabla `clientes`, no repetirse en cada pedido. La tabla `pedidos` solo necesita `cliente_id`.

**Otro ejemplo:** en `employees(id, name, city_id, city_name)`, `city_name` depende de `city_id`, no directamente del empleado. Se crea `cities(id, name)` y `employees` conserva `city_id` como llave foránea.

Para una explicación con tablas, SQL y el proceso completo, consulta la [guía de normalización paso a paso](./06_Normalizacion_Paso_a_Paso.md).

> **💡 Nota Avanzada:** Aunque la normalización es fundamental, en sistemas de alto rendimiento a veces se aplica la **desnormalización** de forma controlada. Esto implica duplicar datos estratégicamente para acelerar las consultas de lectura, evitando `JOINs` complejos a costa de un mayor uso de espacio y una lógica de actualización más cuidadosa.
