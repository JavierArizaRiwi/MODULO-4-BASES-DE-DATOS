# Mejores Prácticas, Seguridad y Backups

Para trabajar con bases de datos en un entorno real y profesional, no basta con saber hacer consultas. Es vital mantener la información segura, rápida y respaldada.

## 1. Convenciones de Nomenclatura
- **Usa `snake_case`:** En PostgreSQL y la mayoría de motores SQL, los nombres de tablas y columnas deben ir en minúsculas y separados por guiones bajos (Ej: `user_roles`, `created_at`).
- **Nombres en plural para las tablas:** Una tabla guarda colecciones de entidades, por lo que debe llamarse `users` o `products`, no `user` o `product`.
- **Evita palabras reservadas:** No uses nombres como `select`, `date`, `user` o `order` solos como nombres de columnas para evitar conflictos con el motor.

## 2. Seguridad: Prevención de Inyección SQL
La Inyección SQL ocurre cuando un usuario malintencionado inserta código SQL dentro de un formulario (como un login) para manipular tu base de datos.

* ❌ **MAL (Vulnerable):** Concatenar strings directamente en el código backend.
  ```javascript
  // Nunca hagas esto en tu código (Ejemplo en Node.js)
  db.query(`SELECT * FROM users WHERE email = '${email}'`);
  ```
* ✅ **BIEN (Seguro):** Usar "Consultas Parametrizadas" (Prepared Statements).
  ```javascript
  // El motor escapará los caracteres peligrosos automáticamente
  db.query('SELECT * FROM users WHERE email = $1', [email]);
  ```

## 3. Uso de Transacciones (TCL)
Si vas a hacer múltiples operaciones que dependen entre sí (como transferir dinero de una cuenta a otra), envuélvelas en una transacción usando `BEGIN` y `COMMIT`. Si algo falla a la mitad, usas `ROLLBACK` y nada se guarda, evitando datos corruptos.

## 4. Backups y Restauración (PostgreSQL)
Aprender a hacer copias de seguridad de tus datos te salvará la vida más de una vez. Utilizando Docker, puedes hacer un backup así:

**Para crear un backup (Respaldo):**
```bash
docker exec -t riwi-postgres pg_dump -U postgres -d riwi_db -F c -f /tmp/backup.dump
``` 

> **💡 Tip de oro:** ¡Nunca pruebes comandos de borrado (`DELETE` o `DROP`) en la base de datos de producción sin haber hecho un `SELECT` antes o sin tener un backup reciente!

## 5. Uso Inteligente de Índices
Los índices aceleran las consultas `SELECT` con cláusulas `WHERE` o `JOIN`, pero ralentizan las operaciones de escritura (`INSERT`, `UPDATE`, `DELETE`).

- **Crea índices en:**
  - Claves foráneas (`FOREIGN KEY`).
  - Columnas que se usan frecuentemente para filtrar datos (Ej: `email` en una tabla de usuarios, `fecha_creacion` en una tabla de logs).
- **No crees índices en:**
  - Columnas con baja cardinalidad (pocos valores únicos, como una columna de "género").
  - Tablas pequeñas, ya que el motor de la base de datos puede leer la tabla completa más rápido de lo que le tomaría usar el índice.