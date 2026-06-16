# Entorno de Desarrollo: PostgreSQL + Docker + DBeaver

Levantar una base de datos local no tiene por qué ser complicado. Utilizando Docker Compose, podemos configurar y encender un servidor de PostgreSQL en cuestión de segundos.

## 1. Archivo docker-compose.yml

Crea un archivo llamado `docker-compose.yml` en la raíz de tu proyecto e inserta esta configuración:

```yaml
services:
  db:
    image: postgres:17
    container_name: riwi-postgres
    environment:
      POSTGRES_DB: riwi_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
    ports:
      - "5433:5432" # Puerto de tu PC : Puerto del contenedor
    volumes:
      - riwi-db-data:/var/lib/postgresql/data # Guarda los datos permanentemente

volumes:
  riwi-db-data:
```

## Comandos

docker compose up -d
docker ps
docker logs riwi-postgres

## DBeaver

Host: localhost
Port: 5433
Database: riwi_db
Username: postgres
Password: postgres123
