# Guía paso a paso: crear un servidor con Express y conectarlo a PostgreSQL usando `pg`

Esta guía explica cómo crear una API básica con **Node.js**, **Express** y la librería **pg** para conectarse a una base de datos PostgreSQL.

La idea es que entiendas el flujo completo:

```text
Cliente / Navegador / Postman
        ↓
Servidor Express
        ↓
Librería pg
        ↓
Base de datos PostgreSQL
```

## 1. ¿Qué vamos a construir?

Vamos a crear un servidor que permita trabajar con estudiantes.

El servidor tendrá endpoints como:

```text
GET    /api/students       → listar estudiantes
GET    /api/students/:id   → buscar un estudiante por id
POST   /api/students       → crear un estudiante
PUT    /api/students/:id   → actualizar un estudiante
DELETE /api/students/:id   → eliminar un estudiante
```

Usaremos la tabla `students`, que ya aparece en la práctica principal del módulo.

## 2. Requisitos previos

Antes de empezar necesitas tener:

- Node.js instalado.
- npm instalado.
- Docker funcionando.
- PostgreSQL levantado con el `docker-compose.yml` del proyecto.
- Un cliente para probar la API, por ejemplo Postman, Insomnia, Thunder Client o el navegador.

Puedes verificar Node.js y npm con:

```bash
node --version
npm --version
```

También verifica que PostgreSQL esté corriendo:

```bash
docker ps
```

Deberías ver un contenedor llamado:

```text
riwi-postgres
```

## 3. Datos de conexión a PostgreSQL

Según el archivo `docker-compose.yml` de este proyecto, la base de datos usa estos datos:

```text
Host: localhost
Port: 5433
Database: riwi_db
User: postgres
Password: postgres123
```

Importante: dentro del contenedor PostgreSQL escucha en el puerto `5432`, pero desde tu computador lo estás exponiendo en el puerto `5433`.

Por eso, desde Node.js usaremos:

```text
localhost:5433
```

## 4. Crear la carpeta del proyecto API

Puedes crear una carpeta separada para la API:

```bash
mkdir api-express-postgres
cd api-express-postgres
```

Inicializa el proyecto Node.js:

```bash
npm init -y
```

Esto crea un archivo `package.json`, que guarda la información del proyecto y sus dependencias.

## 5. Instalar dependencias

Instala Express, pg y dotenv:

```bash
npm install express pg dotenv
```

También puedes instalar `nodemon` para desarrollo:

```bash
npm install -D nodemon
```

¿Para qué sirve cada paquete?

| Paquete | Uso |
|---|---|
| `express` | Crear el servidor y las rutas HTTP. |
| `pg` | Conectarse a PostgreSQL desde Node.js. |
| `dotenv` | Leer variables de entorno desde un archivo `.env`. |
| `nodemon` | Reiniciar el servidor automáticamente cuando cambias el código. |

## 6. Configurar los scripts del proyecto

Abre `package.json` y agrega estos scripts:

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  }
}
```

Ejemplo completo:

```json
{
  "name": "api-express-postgres",
  "version": "1.0.0",
  "description": "API con Express y PostgreSQL",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  },
  "dependencies": {
    "dotenv": "^16.0.0",
    "express": "^4.18.0",
    "pg": "^8.0.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.0"
  }
}
```

No pasa nada si las versiones exactas son diferentes. npm instalará versiones actuales compatibles.

## 7. Crear la estructura de carpetas

Crea esta estructura:

```text
api-express-postgres/
├── src/
│   ├── config/
│   │   └── db.js
│   ├── controllers/
│   │   └── students.controller.js
│   ├── routes/
│   │   └── students.routes.js
│   └── server.js
├── .env
├── .gitignore
└── package.json
```

Puedes crear las carpetas con:

```bash
mkdir -p src/config src/controllers src/routes
```

## 8. Crear el archivo `.env`

El archivo `.env` guarda datos sensibles o configurables.

Crea un archivo llamado `.env`:

```env
PORT=3000

DB_HOST=localhost
DB_PORT=5433
DB_NAME=riwi_db
DB_USER=postgres
DB_PASSWORD=postgres123
```

¿Por qué usar `.env`?

Porque no es buena práctica escribir usuarios, contraseñas o puertos directamente dentro del código.

También crea un archivo `.gitignore`:

```gitignore
node_modules
.env
```

Esto evita subir dependencias y credenciales al repositorio.

## 9. Crear la conexión a PostgreSQL con `pg`

Crea el archivo `src/config/db.js`:

```js
const { Pool } = require('pg');
require('dotenv').config();

const pool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
});

module.exports = pool;
```

### ¿Qué es `Pool`?

`Pool` es un administrador de conexiones.

En lugar de abrir y cerrar una conexión nueva para cada consulta, `pg` mantiene un grupo de conexiones reutilizables.

Ejemplo mental:

```text
Servidor Express
        ↓
Pool de conexiones
        ↓
Conexión 1, Conexión 2, Conexión 3...
        ↓
PostgreSQL
```

Esto es más eficiente que crear una conexión desde cero en cada petición.

## 10. Probar la conexión a la base de datos

Antes de crear todos los endpoints, puedes probar la conexión agregando temporalmente este archivo:

```text
src/test-db.js
```

```js
const pool = require('./config/db');

async function testConnection() {
  try {
    const result = await pool.query('SELECT NOW()');
    console.log('Conexión exitosa:', result.rows[0]);
  } catch (error) {
    console.error('Error conectando a la base de datos:', error.message);
  } finally {
    await pool.end();
  }
}

testConnection();
```

Ejecuta:

```bash
node src/test-db.js
```

Si todo está bien, verás algo parecido a:

```text
Conexión exitosa: { now: 2026-06-24T... }
```

Si aparece un error, revisa:

- Que el contenedor esté encendido.
- Que el puerto sea `5433`.
- Que la contraseña sea `postgres123`.
- Que la base de datos se llame `riwi_db`.

Después de probar, puedes borrar `src/test-db.js` o dejarlo solo como archivo de práctica.

## 11. Crear la tabla `students`

Si aún no tienes la tabla, ejecútala en DBeaver o en `psql`:

```sql
CREATE TABLE IF NOT EXISTS students (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(80) NOT NULL,
    last_name VARCHAR(80) NOT NULL,
    email VARCHAR(120) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Inserta algunos datos de prueba:

```sql
INSERT INTO students (first_name, last_name, email)
VALUES
('Ana', 'Gómez', 'ana.gomez@email.com'),
('Luis', 'Martínez', 'luis.martinez@email.com'),
('Camila', 'Torres', 'camila.torres@email.com');
```

Consulta los datos:

```sql
SELECT * FROM students;
```

## 12. Crear el servidor Express

Crea el archivo `src/server.js`:

```js
const express = require('express');
const studentsRoutes = require('./routes/students.routes');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

app.get('/', (req, res) => {
  res.json({
    message: 'API de estudiantes funcionando correctamente',
  });
});

app.use('/api/students', studentsRoutes);

app.listen(PORT, () => {
  console.log(`Servidor corriendo en http://localhost:${PORT}`);
});
```

### Explicación del archivo

```js
const express = require('express');
```

Importa Express.

```js
const app = express();
```

Crea la aplicación.

```js
app.use(express.json());
```

Permite recibir datos en formato JSON desde el body de las peticiones.

```js
app.use('/api/students', studentsRoutes);
```

Le dice al servidor que todas las rutas de estudiantes empezarán con:

```text
/api/students
```

## 13. Crear las rutas de estudiantes

Crea el archivo `src/routes/students.routes.js`:

```js
const express = require('express');
const {
  getStudents,
  getStudentById,
  createStudent,
  updateStudent,
  deleteStudent,
} = require('../controllers/students.controller');

const router = express.Router();

router.get('/', getStudents);
router.get('/:id', getStudentById);
router.post('/', createStudent);
router.put('/:id', updateStudent);
router.delete('/:id', deleteStudent);

module.exports = router;
```

### ¿Qué hace este archivo?

Este archivo define las rutas, pero no contiene la lógica completa.

Por ejemplo:

```js
router.get('/', getStudents);
```

Significa:

```text
Cuando llegue una petición GET a /api/students,
ejecuta la función getStudents.
```

Separar rutas y controladores ayuda a mantener el código más limpio.

## 14. Crear los controladores

Crea el archivo `src/controllers/students.controller.js`:

```js
const pool = require('../config/db');

const getStudents = async (req, res) => {
  try {
    const result = await pool.query(
      'SELECT id, first_name, last_name, email, created_at FROM students ORDER BY id'
    );

    res.json(result.rows);
  } catch (error) {
    console.error(error);
    res.status(500).json({
      message: 'Error al obtener los estudiantes',
    });
  }
};

const getStudentById = async (req, res) => {
  try {
    const { id } = req.params;

    const result = await pool.query(
      'SELECT id, first_name, last_name, email, created_at FROM students WHERE id = $1',
      [id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({
        message: 'Estudiante no encontrado',
      });
    }

    res.json(result.rows[0]);
  } catch (error) {
    console.error(error);
    res.status(500).json({
      message: 'Error al obtener el estudiante',
    });
  }
};

const createStudent = async (req, res) => {
  try {
    const { first_name, last_name, email } = req.body;

    if (!first_name || !last_name || !email) {
      return res.status(400).json({
        message: 'first_name, last_name y email son obligatorios',
      });
    }

    const result = await pool.query(
      `INSERT INTO students (first_name, last_name, email)
       VALUES ($1, $2, $3)
       RETURNING id, first_name, last_name, email, created_at`,
      [first_name, last_name, email]
    );

    res.status(201).json(result.rows[0]);
  } catch (error) {
    console.error(error);

    if (error.code === '23505') {
      return res.status(409).json({
        message: 'Ya existe un estudiante con ese email',
      });
    }

    res.status(500).json({
      message: 'Error al crear el estudiante',
    });
  }
};

const updateStudent = async (req, res) => {
  try {
    const { id } = req.params;
    const { first_name, last_name, email } = req.body;

    if (!first_name || !last_name || !email) {
      return res.status(400).json({
        message: 'first_name, last_name y email son obligatorios',
      });
    }

    const result = await pool.query(
      `UPDATE students
       SET first_name = $1,
           last_name = $2,
           email = $3
       WHERE id = $4
       RETURNING id, first_name, last_name, email, created_at`,
      [first_name, last_name, email, id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({
        message: 'Estudiante no encontrado',
      });
    }

    res.json(result.rows[0]);
  } catch (error) {
    console.error(error);

    if (error.code === '23505') {
      return res.status(409).json({
        message: 'Ya existe un estudiante con ese email',
      });
    }

    res.status(500).json({
      message: 'Error al actualizar el estudiante',
    });
  }
};

const deleteStudent = async (req, res) => {
  try {
    const { id } = req.params;

    const result = await pool.query(
      'DELETE FROM students WHERE id = $1 RETURNING id, first_name, last_name, email',
      [id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({
        message: 'Estudiante no encontrado',
      });
    }

    res.json({
      message: 'Estudiante eliminado correctamente',
      student: result.rows[0],
    });
  } catch (error) {
    console.error(error);
    res.status(500).json({
      message: 'Error al eliminar el estudiante',
    });
  }
};

module.exports = {
  getStudents,
  getStudentById,
  createStudent,
  updateStudent,
  deleteStudent,
};
```

## 15. Entender `pool.query`

La forma básica de hacer una consulta es:

```js
const result = await pool.query('SELECT * FROM students');
```

El resultado trae varias propiedades, pero la más usada es:

```js
result.rows
```

Ejemplo:

```js
const result = await pool.query('SELECT * FROM students');
console.log(result.rows);
```

Si la tabla tiene tres estudiantes, `result.rows` será un arreglo con tres objetos.

## 16. Consultas parametrizadas

Cuando recibimos datos externos, no debemos concatenarlos directamente en el SQL.

Mal ejemplo:

```js
const result = await pool.query(
  `SELECT * FROM students WHERE id = ${id}`
);
```

Esto puede abrir la puerta a inyección SQL.

Mejor ejemplo:

```js
const result = await pool.query(
  'SELECT * FROM students WHERE id = $1',
  [id]
);
```

Aquí `$1` representa el primer valor del arreglo:

```js
[id]
```

Otro ejemplo con tres valores:

```js
const result = await pool.query(
  `INSERT INTO students (first_name, last_name, email)
   VALUES ($1, $2, $3)
   RETURNING *`,
  [first_name, last_name, email]
);
```

Relación:

| Marcador | Valor |
|---|---|
| `$1` | `first_name` |
| `$2` | `last_name` |
| `$3` | `email` |

## 17. Ejecutar el servidor

En modo desarrollo:

```bash
npm run dev
```

O en modo normal:

```bash
npm start
```

Si todo va bien, verás:

```text
Servidor corriendo en http://localhost:3000
```

Abre en el navegador:

```text
http://localhost:3000
```

Deberías recibir:

```json
{
  "message": "API de estudiantes funcionando correctamente"
}
```

## 18. Probar los endpoints

### Listar estudiantes

Petición:

```http
GET http://localhost:3000/api/students
```

Respuesta esperada:

```json
[
  {
    "id": 1,
    "first_name": "Ana",
    "last_name": "Gómez",
    "email": "ana.gomez@email.com",
    "created_at": "2026-06-24T..."
  }
]
```

### Buscar un estudiante por id

Petición:

```http
GET http://localhost:3000/api/students/1
```

Respuesta esperada:

```json
{
  "id": 1,
  "first_name": "Ana",
  "last_name": "Gómez",
  "email": "ana.gomez@email.com",
  "created_at": "2026-06-24T..."
}
```

Si no existe:

```json
{
  "message": "Estudiante no encontrado"
}
```

### Crear un estudiante

Petición:

```http
POST http://localhost:3000/api/students
Content-Type: application/json
```

Body:

```json
{
  "first_name": "Sofía",
  "last_name": "Ramírez",
  "email": "sofia.ramirez@email.com"
}
```

Respuesta esperada:

```json
{
  "id": 4,
  "first_name": "Sofía",
  "last_name": "Ramírez",
  "email": "sofia.ramirez@email.com",
  "created_at": "2026-06-24T..."
}
```

### Actualizar un estudiante

Petición:

```http
PUT http://localhost:3000/api/students/4
Content-Type: application/json
```

Body:

```json
{
  "first_name": "Sofía",
  "last_name": "Ramírez Castro",
  "email": "sofia.ramirez@email.com"
}
```

Respuesta esperada:

```json
{
  "id": 4,
  "first_name": "Sofía",
  "last_name": "Ramírez Castro",
  "email": "sofia.ramirez@email.com",
  "created_at": "2026-06-24T..."
}
```

### Eliminar un estudiante

Petición:

```http
DELETE http://localhost:3000/api/students/4
```

Respuesta esperada:

```json
{
  "message": "Estudiante eliminado correctamente",
  "student": {
    "id": 4,
    "first_name": "Sofía",
    "last_name": "Ramírez Castro",
    "email": "sofia.ramirez@email.com"
  }
}
```

## 19. Probar con `curl`

También puedes probar desde la terminal.

Listar:

```bash
curl http://localhost:3000/api/students
```

Crear:

```bash
curl -X POST http://localhost:3000/api/students \
  -H "Content-Type: application/json" \
  -d '{"first_name":"Mario","last_name":"Ruiz","email":"mario.ruiz@email.com"}'
```

Actualizar:

```bash
curl -X PUT http://localhost:3000/api/students/1 \
  -H "Content-Type: application/json" \
  -d '{"first_name":"Ana","last_name":"Gómez Pérez","email":"ana.gomez@email.com"}'
```

Eliminar:

```bash
curl -X DELETE http://localhost:3000/api/students/1
```

## 20. Códigos de estado HTTP usados

| Código | Significado | Cuándo lo usamos |
|---|---|---|
| `200` | OK | Consulta, actualización o eliminación exitosa. |
| `201` | Created | Recurso creado correctamente. |
| `400` | Bad Request | Faltan datos obligatorios. |
| `404` | Not Found | El estudiante no existe. |
| `409` | Conflict | El email ya existe. |
| `500` | Internal Server Error | Error inesperado en el servidor. |

## 21. Errores comunes y solución

### Error: `ECONNREFUSED`

Significa que Node.js no pudo conectarse a PostgreSQL.

Revisa:

- Que Docker esté encendido.
- Que el contenedor `riwi-postgres` esté corriendo.
- Que estés usando el puerto `5433`.

### Error: `password authentication failed`

La contraseña es incorrecta.

Revisa el `.env`:

```env
DB_PASSWORD=postgres123
```

### Error: `database "riwi_db" does not exist`

La base de datos no existe o escribiste mal el nombre.

Revisa:

```env
DB_NAME=riwi_db
```

### Error: `relation "students" does not exist`

La tabla `students` no existe.

Solución:

```sql
CREATE TABLE IF NOT EXISTS students (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(80) NOT NULL,
    last_name VARCHAR(80) NOT NULL,
    email VARCHAR(120) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Error: `duplicate key value violates unique constraint`

Estás intentando crear dos estudiantes con el mismo email.

La columna `email` tiene esta restricción:

```sql
email VARCHAR(120) UNIQUE NOT NULL
```

Eso significa:

- `UNIQUE`: no se puede repetir.
- `NOT NULL`: no puede estar vacío.

## 22. Buenas prácticas importantes

### No guardar credenciales directamente en el código

Evita esto:

```js
const pool = new Pool({
  user: 'postgres',
  password: 'postgres123',
});
```

Mejor usa variables de entorno:

```js
const pool = new Pool({
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
});
```

### Usar consultas parametrizadas

Evita concatenar datos enviados por el usuario.

Mal:

```js
`SELECT * FROM students WHERE email = '${email}'`
```

Bien:

```js
'SELECT * FROM students WHERE email = $1'
```

```js
[email]
```

### Separar responsabilidades

Una estructura ordenada ayuda a crecer el proyecto:

| Archivo | Responsabilidad |
|---|---|
| `server.js` | Configura y arranca Express. |
| `db.js` | Crea la conexión a PostgreSQL. |
| `students.routes.js` | Define las rutas. |
| `students.controller.js` | Ejecuta la lógica y las consultas SQL. |

### No mostrar errores internos al usuario

Evita responder errores técnicos completos:

```js
res.status(500).json(error);
```

Mejor:

```js
console.error(error);
res.status(500).json({
  message: 'Error interno del servidor',
});
```

Así el desarrollador ve el error en consola, pero el usuario recibe un mensaje limpio.

## 23. Versión simple en un solo archivo

Para una explicación inicial, también se puede hacer todo en un solo archivo.

Crea `server.js`:

```js
const express = require('express');
const { Pool } = require('pg');

const app = express();
const port = 3000;

app.use(express.json());

const pool = new Pool({
  host: 'localhost',
  port: 5433,
  database: 'riwi_db',
  user: 'postgres',
  password: 'postgres123',
});

app.get('/students', async (req, res) => {
  try {
    const result = await pool.query('SELECT * FROM students ORDER BY id');
    res.json(result.rows);
  } catch (error) {
    console.error(error);
    res.status(500).json({ message: 'Error al consultar estudiantes' });
  }
});

app.listen(port, () => {
  console.log(`Servidor corriendo en http://localhost:${port}`);
});
```

Esta versión sirve para aprender rápido, pero para proyectos reales es mejor separar archivos como hicimos en la guía.

## 24. Reto práctico

Cuando termines la API de estudiantes, crea una API para cursos.

Tabla:

```sql
CREATE TABLE IF NOT EXISTS courses (
    id SERIAL PRIMARY KEY,
    name VARCHAR(120) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Endpoints sugeridos:

```text
GET    /api/courses
GET    /api/courses/:id
POST   /api/courses
PUT    /api/courses/:id
DELETE /api/courses/:id
```

Luego crea un endpoint avanzado:

```text
GET /api/students/:id/courses
```

Ese endpoint debería listar los cursos en los que está inscrito un estudiante.

Para resolverlo necesitarás usar un `JOIN` entre:

- `students`
- `enrollments`
- `courses`

Ejemplo de consulta:

```sql
SELECT
    s.id AS student_id,
    s.first_name,
    s.last_name,
    c.id AS course_id,
    c.name AS course_name,
    e.enrollment_date
FROM students s
INNER JOIN enrollments e ON s.id = e.student_id
INNER JOIN courses c ON c.id = e.course_id
WHERE s.id = $1;
```

## 25. Resumen final

En esta guía aprendiste a:

- Crear un servidor con Express.
- Conectarte a PostgreSQL usando `pg`.
- Configurar variables de entorno con `.env`.
- Crear rutas para una API REST.
- Separar rutas, controladores y configuración.
- Ejecutar consultas SQL desde Node.js.
- Usar consultas parametrizadas para evitar inyección SQL.
- Manejar errores comunes.
- Probar endpoints con navegador, Postman o `curl`.

La idea principal es esta:

```text
Express recibe la petición
        ↓
El controlador procesa la lógica
        ↓
pg ejecuta la consulta SQL
        ↓
PostgreSQL responde
        ↓
Express devuelve JSON al cliente
```

