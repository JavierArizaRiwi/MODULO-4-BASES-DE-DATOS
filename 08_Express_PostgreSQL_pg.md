# Guía fácil: servidor Express conectado a PostgreSQL con `pg`

Esta guía muestra cómo crear una API sencilla con **Node.js**, **Express** y **PostgreSQL**.

La idea es separar el proyecto en pocos archivos:

```text
Cliente / Postman / Thunder Client
        ↓
Servidor Express
        ↓
Repositorio
        ↓
PostgreSQL
```

Vamos a crear dos rutas principales:

```text
GET  /coders    → listar coders
POST /coders    → crear un coder
GET  /          → revisar si la API está activa
```

## 1. Requisitos

Antes de empezar necesitas:

- Node.js instalado.
- npm instalado.
- Docker funcionando.
- PostgreSQL levantado con el `docker-compose.yml` del proyecto.
- Postman, Insomnia, Thunder Client o una herramienta parecida para probar la API.

Verifica Node.js y npm:

```bash
node --version
npm --version
```

Verifica que PostgreSQL esté corriendo:

```bash
docker ps
```

Deberías ver el contenedor de PostgreSQL activo.

## 2. Datos de conexión

En este proyecto PostgreSQL usa estos datos:

```text
Host: localhost
Port: 5433
Database: riwi_db
User: postgres
Password: postgres123
```

Importante: aunque PostgreSQL dentro del contenedor usa el puerto `5432`, desde tu computador lo usas por el puerto `5433`.

## 3. Crear la carpeta del servidor

Crea una carpeta para la API:

```bash
mkdir api-coders
cd api-coders
```

Inicializa el proyecto:

```bash
npm init -y
```

Esto crea el archivo `package.json`.

## 4. Instalar dependencias

Instala Express y pg:

```bash
npm install express pg
```

¿Para qué sirve cada paquete?

| Paquete | Uso |
|---|---|
| `express` | Crear el servidor y las rutas. |
| `pg` | Conectarse a PostgreSQL desde Node.js. |

Opcionalmente instala `nodemon` para desarrollo:

```bash
npm install -D nodemon
```

## 5. Configurar scripts

Abre `package.json` y agrega estos scripts:

```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  }
}
```

Si tu archivo `package.json` ya tiene un bloque `"scripts"`, reemplázalo por ese.

## 6. Crear los archivos del proyecto

El proyecto tendrá esta estructura simple:

```text
api-coders/
├── db.js
├── repositorio.js
├── server.js
└── package.json
```

Crea los archivos:

```bash
touch db.js repositorio.js server.js
```

## 7. Crear la tabla `coders`

Antes de programar el servidor, crea la tabla en PostgreSQL.

Puedes ejecutar este SQL en DBeaver:

```sql
CREATE TABLE IF NOT EXISTS coders (
    id SERIAL PRIMARY KEY,
    tipo_documento VARCHAR(20) NOT NULL,
    numero_documento VARCHAR(30) UNIQUE NOT NULL,
    nombres VARCHAR(80) NOT NULL,
    apellidos VARCHAR(80) NOT NULL,
    correo VARCHAR(120) UNIQUE NOT NULL,
    telefono VARCHAR(30),
    fecha_ingreso DATE NOT NULL,
    id_cohorte INT NOT NULL
);
```

También puedes insertar algunos datos de prueba:

```sql
INSERT INTO coders (
    tipo_documento,
    numero_documento,
    nombres,
    apellidos,
    correo,
    telefono,
    fecha_ingreso,
    id_cohorte
) VALUES
('CC', '1001', 'Ana', 'Gomez', 'ana.gomez@email.com', '3001112233', '2026-01-15', 1),
('CC', '1002', 'Luis', 'Martinez', 'luis.martinez@email.com', '3004445566', '2026-01-15', 1);
```

Revisa que los datos existan:

```sql
SELECT * FROM coders;
```

## 8. Crear la conexión a PostgreSQL

Abre `db.js` y escribe:

```js
const { Pool } = require('pg');

const pool = new Pool({
    host: 'localhost',
    port: 5433,
    database: 'riwi_db',
    user: 'postgres',
    password: 'postgres123',
});

module.exports = pool;
```

### ¿Qué hace este archivo?

Este archivo crea la conexión con PostgreSQL.

`Pool` permite reutilizar conexiones a la base de datos. Así no tienes que abrir una conexión nueva manualmente cada vez que haces una consulta.

## 9. Crear el repositorio

El repositorio será el archivo encargado de hablar con la base de datos.

Abre `repositorio.js` y escribe:

```js
const pool = require('./db');

const getUsers = async () => {
    const result = await pool.query('SELECT * FROM coders ORDER BY id');
    return result.rows;
};

const createUser = async (
    tipo_documento,
    numero_documento,
    nombres,
    apellidos,
    correo,
    telefono,
    fecha_ingreso,
    id_cohorte
) => {
    const result = await pool.query(
        `INSERT INTO coders (
            tipo_documento,
            numero_documento,
            nombres,
            apellidos,
            correo,
            telefono,
            fecha_ingreso,
            id_cohorte
        ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        RETURNING *`,
        [
            tipo_documento,
            numero_documento,
            nombres,
            apellidos,
            correo,
            telefono,
            fecha_ingreso,
            id_cohorte,
        ]
    );

    return result.rows[0];
};

module.exports = {
    getUsers,
    createUser,
};
```

### ¿Qué hace `getUsers`?

Consulta todos los registros de la tabla `coders`:

```js
const result = await pool.query('SELECT * FROM coders ORDER BY id');
return result.rows;
```

`result.rows` contiene los datos que PostgreSQL devuelve.

### ¿Qué hace `createUser`?

Inserta un nuevo coder en la tabla `coders`:

```js
INSERT INTO coders (...) VALUES (...) RETURNING *
```

`RETURNING *` hace que PostgreSQL devuelva el registro recién creado.

## 10. Crear el servidor Express

Abre `server.js` y escribe:

```js
const express = require('express');
const { getUsers, createUser } = require('./repositorio');

const app = express();
const PORT = process.env.PORT || 3001;

app.use(express.json());

app.get('/coders', async (req, res) => {
    try {
        const coders = await getUsers();
        res.json(coders);
    } catch (error) {
        console.error(error);
        res.status(500).json({ error: 'Error al consultar coders' });
    }
});

app.post('/coders', async (req, res) => {
    try {
        const {
            tipo_documento,
            numero_documento,
            nombres,
            apellidos,
            correo,
            telefono,
            fecha_ingreso,
            id_cohorte,
        } = req.body;

        const coder = await createUser(
            tipo_documento,
            numero_documento,
            nombres,
            apellidos,
            correo,
            telefono,
            fecha_ingreso,
            id_cohorte
        );

        res.status(201).json(coder);
    } catch (error) {
        console.error(error);
        res.status(500).json({ error: 'Error al crear coder' });
    }
});

app.get('/', (req, res) => {
    res.send('API de coders activa');
});

app.listen(PORT, () => {
    console.log(`Servidor escuchando en el puerto ${PORT}`);
});
```

## 11. Entender el servidor paso a paso

Importamos Express:

```js
const express = require('express');
```

Importamos las funciones del repositorio:

```js
const { getUsers, createUser } = require('./repositorio');
```

Creamos la aplicación:

```js
const app = express();
```

Definimos el puerto:

```js
const PORT = process.env.PORT || 3001;
```

Activamos la lectura de JSON:

```js
app.use(express.json());
```

Esto permite recibir datos enviados en el body de una petición `POST`.

Creamos la ruta para listar coders:

```js
app.get('/coders', async (req, res) => {
    const coders = await getUsers();
    res.json(coders);
});
```

Creamos la ruta para insertar coders:

```js
app.post('/coders', async (req, res) => {
    const coder = await createUser(...);
    res.status(201).json(coder);
});
```

Creamos una ruta simple para probar el servidor:

```js
app.get('/', (req, res) => {
    res.send('API de coders activa');
});
```

Finalmente levantamos el servidor:

```js
app.listen(PORT, () => {
    console.log(`Servidor escuchando en el puerto ${PORT}`);
});
```

## 12. Ejecutar el servidor

Si instalaste `nodemon`, ejecuta:

```bash
npm run dev
```

Si no instalaste `nodemon`, ejecuta:

```bash
npm start
```

Deberías ver algo parecido a:

```text
Servidor escuchando en el puerto 3001
```

Abre en el navegador:

```text
http://localhost:3001/
```

Deberías ver:

```text
API de coders activa
```

## 13. Probar `GET /coders`

En Postman, Thunder Client o el navegador, prueba:

```text
GET http://localhost:3001/coders
```

Si todo está bien, recibirás un arreglo con los coders:

```json
[
  {
    "id": 1,
    "tipo_documento": "CC",
    "numero_documento": "1001",
    "nombres": "Ana",
    "apellidos": "Gomez",
    "correo": "ana.gomez@email.com",
    "telefono": "3001112233",
    "fecha_ingreso": "2026-01-15T05:00:00.000Z",
    "id_cohorte": 1
  }
]
```

## 14. Probar `POST /coders`

En Postman o Thunder Client:

```text
POST http://localhost:3001/coders
```

En el body selecciona `JSON` y envía:

```json
{
  "tipo_documento": "CC",
  "numero_documento": "1003",
  "nombres": "Camila",
  "apellidos": "Torres",
  "correo": "camila.torres@email.com",
  "telefono": "3007778899",
  "fecha_ingreso": "2026-01-20",
  "id_cohorte": 1
}
```

Si funciona, la API responderá con el coder creado.

## 15. ¿Por qué usamos `$1`, `$2`, `$3`?

En `repositorio.js` usamos una consulta así:

```js
await pool.query(
    'INSERT INTO coders (...) VALUES ($1, $2, $3)',
    [valor1, valor2, valor3]
);
```

Eso se llama consulta parametrizada.

Sirve para enviar datos de forma más segura a PostgreSQL.

No es recomendable hacer esto:

```js
const sql = `SELECT * FROM coders WHERE correo = '${correo}'`;
```

Ese estilo puede provocar errores y problemas de seguridad.

## 16. Errores comunes

### Error de conexión

Si aparece un error como:

```text
ECONNREFUSED
```

Revisa que PostgreSQL esté corriendo:

```bash
docker ps
```

También revisa que el puerto sea `5433`.

### Error porque la tabla no existe

Si aparece:

```text
relation "coders" does not exist
```

Significa que falta crear la tabla `coders`.

Ejecuta el SQL del paso 7.

### Error por correo o documento repetido

Si intentas crear dos coders con el mismo `correo` o `numero_documento`, PostgreSQL puede mostrar un error porque esos campos son únicos.

Cambia esos datos e intenta de nuevo.

## 17. Resumen

En esta guía hiciste lo siguiente:

- Creaste un proyecto Node.js.
- Instalaste Express y pg.
- Creaste una tabla `coders`.
- Conectaste Node.js con PostgreSQL.
- Separaste las consultas en `repositorio.js`.
- Creaste un servidor simple con `server.js`.
- Probaste `GET /coders`.
- Probaste `POST /coders`.

El flujo final queda así:

```text
Petición HTTP
    ↓
server.js
    ↓
repositorio.js
    ↓
db.js
    ↓
PostgreSQL
```
