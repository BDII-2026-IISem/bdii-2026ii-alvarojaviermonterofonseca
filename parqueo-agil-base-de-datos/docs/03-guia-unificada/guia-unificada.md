# Guía unificada: `parqueo_agil_control_de_estacionamiento` y `parqueo_agil_control_de_estacionamientoII`

## Objetivo

Este documento reúne las dos guías anteriores para trabajar con los cuatro motores solicitados:

1. MySQL 8.0
2. PostgreSQL 17
3. Microsoft SQL Server 2022
4. Oracle Database XE 21c

La guía cubre dos instalaciones distintas:

- **Instalación original:** `parqueo_agil_control_de_estacionamiento`. Se reactiva sin crear tablas ni modificar los datos existentes.
- **Instalación II:** `parqueo_agil_control_de_estacionamientoII`. Se crea como una segunda instancia independiente, con sus propios contenedores, puertos, red y almacenamiento.

> Ejecuta los comandos en **Ubuntu/WSL**. No los ejecutes en PowerShell ni en el editor SQL de DBeaver.

## 1. Mapa de las dos instalaciones

| Recurso | Instalación original | Instalación II independiente |
| --- | --- | --- |
| Nombre de base/PDB | `parqueo_agil_control_de_estacionamiento` | `parqueo_agil_control_de_estacionamientoII` |
| Carpeta raíz | `~/ia-lab` | `~/ia-lab-parqueo-agil-ii` |
| Red Docker | `ia-lab-network` | `parqueo-agil-ii-network` |
| MySQL | `mysql-server`, puerto `3306` | `mysql-server-ii`, puerto `3307` |
| PostgreSQL | `postgres-server`, puerto `5432` | `postgres-server-ii`, puerto `5433` |
| SQL Server | `mssql-server`, puerto `1433` | `mssql-server-ii`, puerto `1434` |
| Oracle XE | `oracle-server`, puerto `1521` | `oracle-server-ii`, puerto `1522` |

Las dos instalaciones pueden funcionar simultáneamente porque no comparten nombre de contenedor, red, ruta de datos ni puerto del equipo.

## 2. Reglas importantes

- No elimines `~/ia-lab/data/`: contiene los datos de la instalación original.
- No ejecutes `docker compose down -v` sobre la instalación original ni sobre la II si quieres conservar su información.
- No vuelvas a ejecutar scripts `CREATE TABLE` en la base original si sus tablas ya existen.
- La instalación II es nueva: levantarla no copia las tablas de la instalación original.
- Los archivos `.env` contienen contraseñas y no deben subirse a Git ni compartirse.

## 3. Verificación inicial de Docker

```bash
sudo systemctl start docker
sudo systemctl is-active docker
docker --version
docker compose version
sudo docker ps -a
```

El resultado esperado de `sudo systemctl is-active docker` es `active`.

---

# 4. MySQL 8.0

## 4.1 Reactivar la base original

La base original está en `~/ia-lab/services/motores-bd/mysql` y utiliza el contenedor `mysql-server`.

```bash
cd ~/ia-lab/services/motores-bd/mysql
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker logs mysql-server --tail 30
```

Verifica que la base y sus tablas sigan disponibles. El comando solicitará la contraseña de `root` definida en el `.env` original:

```bash
sudo docker exec -it mysql-server mysql -u root -p
```

```sql
SHOW DATABASES;
USE parqueo_agil_control_de_estacionamiento;
SHOW TABLES;
EXIT;
```

### DBeaver: base original MySQL

| Campo | Valor |
| --- | --- |
| Driver | MySQL |
| Host | `localhost` |
| Port | `3306` |
| Database | `parqueo_agil_control_de_estacionamiento` |
| Username | `admin` si existe; de lo contrario, `root` |
| Password | clave correspondiente del `.env` original |

Pulsa **Test Connection** y luego **Finish**.

## 4.2 Crear la instalación II de MySQL

### Crear las carpetas y la red II

> Ejecuta esta preparación una sola vez; es compartida por los cuatro motores de la instalación II.

```bash
mkdir -p ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/{mysql,postgres,mssql,oracle}
mkdir -p ~/ia-lab-parqueo-agil-ii/data/{mysql,postgres,mssql}
mkdir -p ~/ia-lab-parqueo-agil-ii/backups

sudo docker network inspect parqueo-agil-ii-network >/dev/null 2>&1 || \
sudo docker network create parqueo-agil-ii-network
```

### Crear `docker-compose.yml`

```bash
cat > ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mysql/docker-compose.yml <<'EOF'
services:
  mysql:
    image: mysql:8.0
    container_name: mysql-server-ii
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "3307:3306"
    volumes:
      - ../../../data/mysql:/var/lib/mysql
      - ../../../backups:/backups
    command: >
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --bind-address=0.0.0.0
    networks:
      - parqueo-agil-ii-network
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -u root -p$$MYSQL_ROOT_PASSWORD || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

networks:
  parqueo-agil-ii-network:
    external: true
EOF
```

### Crear `.env`

Reemplaza la contraseña de ejemplo antes de iniciar el servicio.

```bash
cat > ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mysql/.env <<'EOF'
TZ=America/Bogota
MYSQL_ROOT_PASSWORD=<TU_CONTRASENA_SEGURA>
MYSQL_DATABASE=parqueo_agil_control_de_estacionamientoII
MYSQL_USER=admin
MYSQL_PASSWORD=<TU_CONTRASENA_SEGURA>
EOF
```

### Levantar MySQL II

```bash
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mysql
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker logs mysql-server-ii --tail 30
```

### DBeaver: MySQL II

| Campo | Valor |
| --- | --- |
| Driver | MySQL |
| Host | `localhost` |
| Port | `3307` |
| Database | `parqueo_agil_control_de_estacionamientoII` |
| Username | `admin` |
| Password | valor de `MYSQL_PASSWORD` |

---

# 5. PostgreSQL 17

## 5.1 Reactivar la base original

```bash
cd ~/ia-lab/services/motores-bd/postgres
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker logs postgres-server --tail 30
```

La guía original utiliza el usuario `admin` para trabajar en DBeaver. Comprueba la conexión y las tablas:

```bash
sudo docker exec -it postgres-server \
  psql -U admin -d parqueo_agil_control_de_estacionamiento \
  -c 'SELECT current_database(), current_user;'

sudo docker exec -it postgres-server \
  psql -U admin -d parqueo_agil_control_de_estacionamiento \
  -c '\dt'
```

Si `admin` no existe, revisa los usuarios iniciales con:

```bash
sudo docker exec -it postgres-server psql -U postgres -d postgres -c '\du'
```

### DBeaver: base original PostgreSQL

| Campo | Valor |
| --- | --- |
| Driver | PostgreSQL |
| Host | `localhost` |
| Port | `5432` |
| Database | `parqueo_agil_control_de_estacionamiento` |
| Username | `admin` |
| Password | contraseña original de `admin` |

## 5.2 Crear la instalación II de PostgreSQL

### Crear `docker-compose.yml`

```bash
cat > ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/postgres/docker-compose.yml <<'EOF'
services:
  postgres:
    image: postgres:17
    container_name: postgres-server-ii
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "5433:5432"
    volumes:
      - ../../../data/postgres:/var/lib/postgresql/data
      - ../../../backups:/backups
    command:
      - postgres
      - -c
      - listen_addresses=*
    networks:
      - parqueo-agil-ii-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

networks:
  parqueo-agil-ii-network:
    external: true
EOF
```

### Crear `.env`

```bash
cat > ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/postgres/.env <<'EOF'
TZ=America/Bogota
POSTGRES_USER=admin
POSTGRES_PASSWORD=<TU_CONTRASENA_SEGURA>
POSTGRES_DB=parqueo_agil_control_de_estacionamientoII
EOF
```

`POSTGRES_DB` y `POSTGRES_USER` se aplican solo cuando `data/postgres` está vacío.

### Levantar PostgreSQL II

```bash
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/postgres
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker logs postgres-server-ii --tail 30

sudo docker exec -it postgres-server-ii \
  psql -U admin -d parqueo_agil_control_de_estacionamientoII \
  -c 'SELECT current_database(), current_user;'
```

### DBeaver: PostgreSQL II

| Campo | Valor |
| --- | --- |
| Driver | PostgreSQL |
| Host | `localhost` |
| Port | `5433` |
| Database | `parqueo_agil_control_de_estacionamientoII` |
| Username | `admin` |
| Password | valor de `POSTGRES_PASSWORD` |

---

# 6. Microsoft SQL Server 2022

## 6.1 Reactivar la base original

```bash
cd ~/ia-lab/services/motores-bd/mssql
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker logs mssql-server --tail 40
```

Espera a que el registro indique que SQL Server está listo para conexiones de cliente. La contraseña de `SA` se encuentra en `~/ia-lab/services/motores-bd/mssql/.env`. Sustituye `<CONTRASENA_SA>`:

```bash
sudo docker exec -it mssql-server /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U SA -P '<CONTRASENA_SA>' -C -d master \
  -Q "SELECT name FROM sys.databases WHERE name = N'parqueo_agil_control_de_estacionamiento';"
```

### DBeaver: base original SQL Server

| Campo | Valor |
| --- | --- |
| Driver | SQL Server |
| Host | `localhost` |
| Port | `1433` |
| Database | `parqueo_agil_control_de_estacionamiento` |
| Username | `SA` |
| Password | valor de `MSSQL_SA_PASSWORD` del `.env` original |

Si DBeaver presenta un error de certificado, activa **Trust server certificate** en SSL o Driver properties.

## 6.2 Crear la instalación II de SQL Server

### Crear `docker-compose.yml`

```bash
cat > ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mssql/docker-compose.yml <<'EOF'
services:
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: mssql-server-ii
    restart: unless-stopped
    user: root
    env_file:
      - .env
    ports:
      - "1434:1433"
    volumes:
      - ../../../data/mssql:/var/opt/mssql
      - ../../../backups:/backups
    networks:
      - parqueo-agil-ii-network

networks:
  parqueo-agil-ii-network:
    external: true
EOF
```

### Crear `.env`

```bash
cat > ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mssql/.env <<'EOF'
TZ=America/Bogota
ACCEPT_EULA=Y
MSSQL_SA_PASSWORD=<TU_CONTRASENA_SEGURA>
MSSQL_PID=Developer
EOF
```

La contraseña de `SA` debe incluir mayúscula, minúscula, número y símbolo. Defínela correctamente antes del primer inicio.

### Levantar SQL Server II y crear la base

```bash
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mssql
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker logs mssql-server-ii --tail 40
```

Una vez listo el motor, crea la base II solo una vez:

```bash
sudo docker exec -it mssql-server-ii /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U SA -P '<TU_CONTRASENA_SEGURA>' -C -d master \
  -Q "IF DB_ID(N'parqueo_agil_control_de_estacionamientoII') IS NULL BEGIN EXEC(N'CREATE DATABASE [parqueo_agil_control_de_estacionamientoII]'); END"
```

Si cambiaste la clave del `.env`, reemplázala también en este comando.

### DBeaver: SQL Server II

| Campo | Valor |
| --- | --- |
| Driver | SQL Server |
| Host | `localhost` |
| Port | `1434` |
| Database | `parqueo_agil_control_de_estacionamientoII` |
| Username | `SA` |
| Password | valor de `MSSQL_SA_PASSWORD` |

---

# 7. Oracle Database XE 21c

## 7.1 Reactivar la base original

```bash
cd ~/ia-lab/services/motores-bd/oracle
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker logs -f oracle-server
```

Oracle puede tardar varios minutos en estar disponible. Cuando los registros indiquen que está listo, presiona `Ctrl + C`; solo detiene la visualización de registros, no el contenedor.

La base original usa el PDB/Service Name `parqueo_agil_control_de_estacionamiento` y las tablas pertenecen al usuario `admin`.

### DBeaver: base original Oracle

| Campo | Valor |
| --- | --- |
| Driver | Oracle |
| Host | `localhost` |
| Port | `1521` |
| Connection type | `Basic` |
| Service Name | `parqueo_agil_control_de_estacionamiento` |
| Username | `admin` |
| Password | contraseña original de `admin` |
| Role | `Normal` |

Selecciona **Service Name**, no SID.

## 7.2 Crear la instalación II de Oracle XE

### Crear `docker-compose.yml`

Oracle II utiliza un volumen Docker con nombre para evitar los problemas de permisos que pueden aparecer al montar una carpeta de WSL en `/opt/oracle/oradata`.

```bash
cat > ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/oracle/docker-compose.yml <<'EOF'
services:
  oracle:
    image: gvenzl/oracle-xe:21-slim
    container_name: oracle-server-ii
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "1522:1521"
    volumes:
      - oracle_ii_data:/opt/oracle/oradata
      - ../../../backups:/backups
    networks:
      - parqueo-agil-ii-network

volumes:
  oracle_ii_data:
    name: parqueo_agil_oracle_ii_data

networks:
  parqueo-agil-ii-network:
    external: true
EOF
```

No agregues `command:` al archivo de Oracle, porque puede reemplazar el arranque definido por la imagen.

### Crear `.env`

```bash
cat > ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/oracle/.env <<'EOF'
TZ=America/Bogota
ORACLE_PASSWORD=<TU_CONTRASENA_SEGURA>
ORACLE_DATABASE=parqueo_agil_control_de_estacionamientoII
EOF
```

`ORACLE_DATABASE` es el PDB y el Service Name. No es un usuario.

### Levantar Oracle II

```bash
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/oracle
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker logs -f oracle-server-ii
```

### Crear el esquema de trabajo II

Cuando el contenedor esté listo, entra y crea el usuario que será propietario de las tablas:

```bash
sudo docker exec -it oracle-server-ii bash
sqlplus 'system/<TU_CONTRASENA_SEGURA>@//localhost:1521/parqueo_agil_control_de_estacionamientoII'
```

Al aparecer `SQL>`, ejecuta:

```sql
CREATE USER admin IDENTIFIED BY "Admin_II_2026!" DEFAULT TABLESPACE USERS QUOTA UNLIMITED ON USERS;
GRANT CREATE SESSION, CREATE TABLE, CREATE SEQUENCE, CREATE VIEW, CREATE PROCEDURE, CREATE TRIGGER TO admin;
EXIT;
```

Luego escribe `exit` para salir del contenedor. Si el usuario ya existe por una restauración previa, no ejecutes de nuevo `CREATE USER`.

### DBeaver: Oracle II

| Campo | Valor |
| --- | --- |
| Driver | Oracle |
| Host | `localhost` |
| Port | `1522` |
| Connection type | `Basic` |
| Service Name | `parqueo_agil_control_de_estacionamientoII` |
| Username | `admin` |
| Password | `Admin_II_2026!` o la clave que elegiste |
| Role | `Normal` |

---

# 8. Tablas y datos de la instalación II

La instalación II comienza vacía. Para que contenga las tablas que ya existen en la base original, elige una de estas opciones:

1. Ejecutar los archivos `.sql` de creación de tablas en cada nueva conexión II desde DBeaver.
2. Exportar la estructura y/o los datos desde la base original e importarlos en la base II.
3. Restaurar un respaldo que corresponda al motor y a la nueva instancia.

Después de importar o crear las tablas, verifica desde DBeaver.

### MySQL

```sql
SHOW TABLES;
```

### PostgreSQL

```sql
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
  AND table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name;
```

### SQL Server

```sql
SELECT name FROM sys.tables ORDER BY name;
```

### Oracle

```sql
SELECT table_name FROM user_tables ORDER BY table_name;
```

# 9. Pasar la información de la base original a la base II

## 9.1 Qué transfiere este procedimiento

Esta sección migra la estructura y los datos de la instalación original hacia la instalación II **dentro del mismo motor**:

```text
parqueo_agil_control_de_estacionamiento
                │
                │  exportación / respaldo
                ▼
parqueo_agil_control_de_estacionamientoII
```

Por ejemplo, la copia de MySQL solo se restaura en MySQL II; la de PostgreSQL solo se restaura en PostgreSQL II. Los formatos de respaldo no son intercambiables entre motores.

La base original no se elimina ni se altera. Las instrucciones de restauración reemplazan las tablas y los datos del destino II cuando ya existen. Ejecuta cada bloque solo después de confirmar que los dos contenedores del motor correspondiente estén activos.

Antes de iniciar, crea o verifica el directorio común de respaldos de la instalación II:

```bash
mkdir -p ~/ia-lab-parqueo-agil-ii/backups
ls -lh ~/ia-lab-parqueo-agil-ii/backups
```

## 9.2 MySQL: original → MySQL II

### Paso 1. Confirmar los contenedores

```bash
sudo docker ps --filter 'name=mysql-server' --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Debes ver `mysql-server` y `mysql-server-ii`. Si alguno no aparece, levántalo con las secciones 4.1 o 4.2 de esta guía.

### Paso 2. Exportar estructura y datos de la base original

Este comando crea un archivo SQL en el equipo. Lee la contraseña del usuario `root` desde las variables del contenedor original, por lo que no debes escribirla en la consola.

```bash
sudo docker exec mysql-server sh -c \
  'exec mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" \
  --single-transaction --routines --triggers --events \
  parqueo_agil_control_de_estacionamiento' \
  > ~/ia-lab-parqueo-agil-ii/backups/mysql_original_a_ii.sql
```

Comprueba que el respaldo no esté vacío:

```bash
ls -lh ~/ia-lab-parqueo-agil-ii/backups/mysql_original_a_ii.sql
```

### Paso 3. Preparar la base de destino II

> **Atención:** este comando elimina únicamente la base MySQL II y la crea de nuevo vacía. No toca la base original.

```bash
sudo docker exec mysql-server-ii sh -c \
  'exec mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "DROP DATABASE IF EXISTS parqueo_agil_control_de_estacionamientoII; CREATE DATABASE parqueo_agil_control_de_estacionamientoII CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci; GRANT ALL PRIVILEGES ON parqueo_agil_control_de_estacionamientoII.* TO '\''admin'\''@'\''%'\''; FLUSH PRIVILEGES;"'
```

### Paso 4. Importar en MySQL II

```bash
sudo docker exec -i mysql-server-ii sh -c \
  'exec mysql -u root -p"$MYSQL_ROOT_PASSWORD" parqueo_agil_control_de_estacionamientoII' \
  < ~/ia-lab-parqueo-agil-ii/backups/mysql_original_a_ii.sql
```

### Paso 5. Verificar desde MySQL II o DBeaver

```bash
sudo docker exec mysql-server-ii sh -c \
  'exec mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "USE parqueo_agil_control_de_estacionamientoII; SHOW TABLES;"'
```

En DBeaver abre la conexión **MySQL II**: host `localhost`, puerto `3307`, base `parqueo_agil_control_de_estacionamientoII` y usuario `admin`.

## 9.3 PostgreSQL: original → PostgreSQL II

### Paso 1. Confirmar los contenedores

```bash
sudo docker ps --filter 'name=postgres-server' --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Debes ver `postgres-server` y `postgres-server-ii`.

### Paso 2. Exportar la base original

La instalación original se inicializó con el superusuario `postgres`. El respaldo contiene estructura, secuencias, índices, vistas y datos, pero evita copiar propietarios y privilegios que pertenecen a la instancia anterior.

```bash
sudo docker exec postgres-server \
  pg_dump -U postgres \
  --format=plain --clean --if-exists --no-owner --no-privileges \
  --file=/tmp/postgres_original_a_ii.sql \
  parqueo_agil_control_de_estacionamiento

sudo docker cp postgres-server:/tmp/postgres_original_a_ii.sql \
  ~/ia-lab-parqueo-agil-ii/backups/postgres_original_a_ii.sql

ls -lh ~/ia-lab-parqueo-agil-ii/backups/postgres_original_a_ii.sql
```

### Paso 3. Restaurar en PostgreSQL II

`--clean` hace que el archivo elimine los objetos del destino antes de crearlos otra vez. Por tanto, puede ejecutarse para reemplazar los datos previos de la base II, sin tocar la original.

```bash
sudo docker exec -i postgres-server-ii \
  psql -v ON_ERROR_STOP=1 -U admin \
  -d parqueo_agil_control_de_estacionamientoII \
  < ~/ia-lab-parqueo-agil-ii/backups/postgres_original_a_ii.sql
```

### Paso 4. Verificar desde PostgreSQL II o DBeaver

```bash
sudo docker exec postgres-server-ii \
  psql -U admin -d parqueo_agil_control_de_estacionamientoII \
  -c '\dt'
```

En DBeaver abre la conexión **PostgreSQL II**: host `localhost`, puerto `5433`, base `parqueo_agil_control_de_estacionamientoII` y usuario `admin`.

## 9.4 SQL Server: original → SQL Server II

### Paso 1. Definir las contraseñas solo para esta sesión

Lee las contraseñas de ambos archivos `.env`. No las escribas en el historial de comandos; usa los valores reales cuando la terminal los solicite o reemplaza las variables en el bloque siguiente.

```bash
cat ~/ia-lab/services/motores-bd/mssql/.env
cat ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mssql/.env
```

Define variables temporales en la terminal. Reemplaza cada valor antes de presionar Enter:

```bash
read -s -p 'Contraseña de SA original: ' MSSQL_ORIGINAL_SA_PASSWORD; echo
read -s -p 'Contraseña de SA II: ' MSSQL_II_SA_PASSWORD; echo
```

### Paso 2. Crear un respaldo de la base original

```bash
sudo docker exec mssql-server /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U SA -P "$MSSQL_ORIGINAL_SA_PASSWORD" -C -d master \
  -Q "BACKUP DATABASE [parqueo_agil_control_de_estacionamiento] TO DISK = N'/backups/mssql_original_a_ii.bak' WITH INIT, COPY_ONLY, COMPRESSION, STATS = 10;"
```

El contenedor original tiene montada la ruta `/backups`. Copia el archivo desde ese contenedor hasta el directorio de respaldos de la instalación II:

```bash
sudo docker cp mssql-server:/backups/mssql_original_a_ii.bak \
  ~/ia-lab-parqueo-agil-ii/backups/mssql_original_a_ii.bak

ls -lh ~/ia-lab-parqueo-agil-ii/backups/mssql_original_a_ii.bak
```

### Paso 3. Copiar el respaldo al contenedor II

```bash
sudo docker exec mssql-server-ii mkdir -p /var/opt/mssql/backup

sudo docker cp \
  ~/ia-lab-parqueo-agil-ii/backups/mssql_original_a_ii.bak \
  mssql-server-ii:/var/opt/mssql/backup/mssql_original_a_ii.bak
```

### Paso 4. Verificar los nombres lógicos del respaldo

Normalmente, si la base se creó con `CREATE DATABASE parqueo_agil_control_de_estacionamiento`, los nombres lógicos son los utilizados en el paso siguiente. Ejecuta esta consulta para confirmarlo:

```bash
sudo docker exec -it mssql-server-ii /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U SA -P "$MSSQL_II_SA_PASSWORD" -C -d master \
  -Q "RESTORE FILELISTONLY FROM DISK = N'/var/opt/mssql/backup/mssql_original_a_ii.bak';"
```

Confirma que aparecen los nombres lógicos:

```text
parqueo_agil_control_de_estacionamiento
parqueo_agil_control_de_estacionamiento_log
```

Si el resultado muestra nombres distintos, sustituye solo los dos valores que aparecen después de `MOVE N'...'` en el paso 5.

### Paso 5. Reemplazar la base II y restaurar el respaldo

> **Atención:** este bloque elimina únicamente `parqueo_agil_control_de_estacionamientoII` dentro de SQL Server II antes de restaurarla. Nunca se conecta al contenedor original.

```bash
sudo docker exec -i mssql-server-ii /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U SA -P "$MSSQL_II_SA_PASSWORD" -C -d master <<'SQL'
IF DB_ID(N'parqueo_agil_control_de_estacionamientoII') IS NOT NULL
BEGIN
    ALTER DATABASE [parqueo_agil_control_de_estacionamientoII]
    SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
    DROP DATABASE [parqueo_agil_control_de_estacionamientoII];
END
GO

RESTORE DATABASE [parqueo_agil_control_de_estacionamientoII]
FROM DISK = N'/var/opt/mssql/backup/mssql_original_a_ii.bak'
WITH
    MOVE N'parqueo_agil_control_de_estacionamiento'
        TO N'/var/opt/mssql/data/parqueo_agil_control_de_estacionamientoII.mdf',
    MOVE N'parqueo_agil_control_de_estacionamiento_log'
        TO N'/var/opt/mssql/data/parqueo_agil_control_de_estacionamientoII_log.ldf',
    RECOVERY,
    STATS = 10;
GO
SQL
```

### Paso 6. Verificar desde SQL Server II o DBeaver

```bash
sudo docker exec -it mssql-server-ii /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U SA -P "$MSSQL_II_SA_PASSWORD" -C \
  -d parqueo_agil_control_de_estacionamientoII \
  -Q "SELECT name FROM sys.tables ORDER BY name;"

unset MSSQL_ORIGINAL_SA_PASSWORD MSSQL_II_SA_PASSWORD
```

En DBeaver abre la conexión **SQL Server II**: host `localhost`, puerto `1434`, base `parqueo_agil_control_de_estacionamientoII` y usuario `SA`.

## 9.5 Oracle XE: original → Oracle II

Oracle utiliza PDB para la base y un usuario/esquema propietario de las tablas. En esta guía se exporta e importa el esquema `admin`.

### Paso 1. Confirmar que existe el usuario `admin` en Oracle II

La instalación II debe estar encendida y tener su usuario `admin` creado según la sección 7.2. Si aún no lo creaste, ejecuta primero esa sección.

### Paso 2. Solicitar las contraseñas sin dejarlas en el historial

```bash
read -s -p 'Contraseña de admin en Oracle original: ' ORACLE_ORIGINAL_ADMIN_PASSWORD; echo
read -s -p 'Contraseña de admin en Oracle II: ' ORACLE_II_ADMIN_PASSWORD; echo
```

### Paso 3. Exportar el esquema `admin` de Oracle original

```bash
sudo docker exec oracle-server expdp \
  "admin/$ORACLE_ORIGINAL_ADMIN_PASSWORD@//localhost:1521/parqueo_agil_control_de_estacionamiento" \
  directory=DATA_PUMP_DIR \
  dumpfile=oracle_original_a_ii.dmp \
  logfile=oracle_original_a_ii_export.log \
  schemas=ADMIN
```

La imagen utiliza el directorio Data Pump de XE. Copia el respaldo y su registro al equipo:

```bash
sudo docker cp oracle-server:/opt/oracle/admin/XE/dpdump/oracle_original_a_ii.dmp \
  ~/ia-lab-parqueo-agil-ii/backups/oracle_original_a_ii.dmp

sudo docker cp oracle-server:/opt/oracle/admin/XE/dpdump/oracle_original_a_ii_export.log \
  ~/ia-lab-parqueo-agil-ii/backups/oracle_original_a_ii_export.log

ls -lh ~/ia-lab-parqueo-agil-ii/backups/oracle_original_a_ii.*
```

### Paso 4. Copiar el respaldo al contenedor Oracle II

```bash
sudo docker cp \
  ~/ia-lab-parqueo-agil-ii/backups/oracle_original_a_ii.dmp \
  oracle-server-ii:/opt/oracle/admin/XE/dpdump/oracle_original_a_ii.dmp
```

### Paso 5. Importar el esquema en Oracle II

`TABLE_EXISTS_ACTION=REPLACE` hace que las tablas existentes del esquema `admin` en Oracle II sean reemplazadas. La instancia y PDB originales no son modificados.

```bash
sudo docker exec oracle-server-ii impdp \
  "admin/$ORACLE_II_ADMIN_PASSWORD@//localhost:1521/parqueo_agil_control_de_estacionamientoII" \
  directory=DATA_PUMP_DIR \
  dumpfile=oracle_original_a_ii.dmp \
  logfile=oracle_original_a_ii_import.log \
  schemas=ADMIN \
  table_exists_action=REPLACE
```

### Paso 6. Verificar desde Oracle II o DBeaver

```bash
sudo docker exec oracle-server-ii sqlplus -s \
  "admin/$ORACLE_II_ADMIN_PASSWORD@//localhost:1521/parqueo_agil_control_de_estacionamientoII" <<'SQL'
SET PAGESIZE 200
SELECT table_name FROM user_tables ORDER BY table_name;
EXIT;
SQL

unset ORACLE_ORIGINAL_ADMIN_PASSWORD ORACLE_II_ADMIN_PASSWORD
```

En DBeaver abre la conexión **Oracle II**: host `localhost`, puerto `1522`, selecciona **Service Name**, escribe `parqueo_agil_control_de_estacionamientoII`, usuario `admin` y rol `Normal`.

## 9.6 Validación final de la migración

Después de cada copia, compara en DBeaver la cantidad de tablas de la original y de la II. También selecciona algunas filas de las tablas principales para confirmar que los datos fueron transferidos.

Ejemplo general para una tabla conocida:

```sql
SELECT COUNT(*) AS total_registros
FROM nombre_de_tu_tabla;
```

Ejecuta la misma consulta en la conexión original y en la conexión II del mismo motor. Los resultados deben coincidir cuando no existen cambios realizados en la base original durante la migración.

## 9.7 Pasar información entre motores distintos

No uses los archivos `.sql`, `.bak` o `.dmp` anteriores para mover una base entre MySQL, PostgreSQL, SQL Server y Oracle: cada motor tiene formatos, tipos de datos, secuencias, identidades y sintaxis diferentes.

Para una migración entre motores distintos se necesita un proceso separado: convertir el esquema al motor destino, importar los datos en orden de dependencias y validar claves, fechas, decimales y caracteres. Las conexiones de DBeaver permiten exportar e importar datos, pero no convierten automáticamente todo el modelo sin revisión.

# 10. Operación, diagnóstico y acceso remoto

## Ver todos los contenedores y puertos

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Puertos de la instalación original: `3306`, `5432`, `1433`, `1521`.

Puertos de la instalación II: `3307`, `5433`, `1434`, `1522`.

## Ver registros

```bash
sudo docker logs mysql-server --tail 50
sudo docker logs postgres-server --tail 50
sudo docker logs mssql-server --tail 50
sudo docker logs oracle-server --tail 50

sudo docker logs mysql-server-ii --tail 50
sudo docker logs postgres-server-ii --tail 50
sudo docker logs mssql-server-ii --tail 50
sudo docker logs oracle-server-ii --tail 50
```

## Iniciar todos los contenedores de una instalación

### Instalación original

```bash
cd ~/ia-lab/services/motores-bd/mysql && sudo docker compose up -d
cd ~/ia-lab/services/motores-bd/postgres && sudo docker compose up -d
cd ~/ia-lab/services/motores-bd/mssql && sudo docker compose up -d
cd ~/ia-lab/services/motores-bd/oracle && sudo docker compose up -d
```

### Instalación II

```bash
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mysql && sudo docker compose up -d
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/postgres && sudo docker compose up -d
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mssql && sudo docker compose up -d
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/oracle && sudo docker compose up -d
```

## DBeaver no conecta

1. Confirma que el contenedor está `running` con `sudo docker ps`.
2. Revisa los últimos registros del contenedor.
3. Confirma que seleccionaste el puerto correcto: original o II.
4. Revisa usuario, contraseña, base y Service Name.
5. Si DBeaver se ejecuta en el mismo computador, usa `localhost`.
6. Para SQL Server, activa **Trust server certificate** si el certificado local genera error.

## Conexión desde otro equipo de la red

Sustituye `localhost` por la IP del equipo anfitrión y abre solo el puerto que necesites:

```bash
sudo ufw allow 3306/tcp
sudo ufw allow 5432/tcp
sudo ufw allow 1433/tcp
sudo ufw allow 1521/tcp

sudo ufw allow 3307/tcp
sudo ufw allow 5433/tcp
sudo ufw allow 1434/tcp
sudo ufw allow 1522/tcp
sudo ufw status
```

No abras los puertos si utilizarás DBeaver solo desde el mismo computador y nunca los expongas en una red pública.

## Resultado esperado

Al finalizar tendrás ocho conexiones posibles en DBeaver: cuatro para la instalación original y cuatro para la instalación II. Cada conexión será independiente según su motor, base, contenedor y puerto.

# 11. Cambiar de una base a otra en DBeaver y transferencia visual

## 11.1 Dos acciones distintas

La expresión “pasar de una base a otra” puede significar dos cosas diferentes:

1. **Cambiar de conexión:** abrir en DBeaver la base original o la base II para trabajar en una u otra. Esto **no copia, modifica ni borra datos**.
2. **Migrar datos:** copiar la estructura de las tablas y/o sus registros desde la base original hacia la base II. Esto es una operación adicional y deliberada.

La base original y la base II son independientes. Nada de la primera aparece automáticamente en la segunda.

## 11.2 Cambiar de conexión en DBeaver, sin mover datos

Conserva ambas conexiones en DBeaver y ponles nombres claros. Por ejemplo:

| Motor | Conexión original | Conexión II |
| --- | --- | --- |
| MySQL | `ParqueoAgil - MySQL - Original (3306)` | `ParqueoAgil - MySQL - II (3307)` |
| PostgreSQL | `ParqueoAgil - PostgreSQL - Original (5432)` | `ParqueoAgil - PostgreSQL - II (5433)` |
| SQL Server | `ParqueoAgil - SQL Server - Original (1433)` | `ParqueoAgil - SQL Server - II (1434)` |
| Oracle | `ParqueoAgil - Oracle - Original (1521)` | `ParqueoAgil - Oracle - II (1522)` |

Para pasar de una conexión a la otra:

1. En el panel **Database Navigator** de DBeaver, expande la conexión que quieres usar.
2. Haz doble clic sobre ella, si todavía no está conectada.
3. Haz clic derecho sobre el nombre de la base y selecciona **SQL Editor > New SQL Script**.
4. Confirma en la parte superior del editor que el script está asociado a la conexión correcta antes de ejecutar consultas o cambios.

Usa estas consultas para comprobar dónde estás trabajando:

### MySQL

```sql
SELECT DATABASE() AS base_actual;
```

### PostgreSQL

```sql
SELECT current_database() AS base_actual, current_user AS usuario_actual;
```

### SQL Server

```sql
SELECT DB_NAME() AS base_actual, SUSER_SNAME() AS usuario_actual;
```

### Oracle

```sql
SELECT SYS_CONTEXT('USERENV', 'SERVICE_NAME') AS servicio_actual,
       USER AS usuario_actual
FROM dual;
```

Si el resultado muestra `parqueo_agil_control_de_estacionamiento`, estás en la instalación original. Si muestra `parqueo_agil_control_de_estacionamientoII`, estás en la nueva instalación.

## 11.3 Alternativa visual: migrar tablas y datos con DBeaver

Antes de transferir información, define qué necesitas llevar:

- **Solo estructura:** tablas, llaves, índices, vistas y procedimientos, pero sin registros.
- **Solo datos:** registros para tablas que ya existen y son compatibles en la base II.
- **Estructura y datos:** una copia completa del proyecto, que es el caso más frecuente.

La forma más segura es migrar primero la estructura y luego los registros. La base II debe estar vacía o contener tablas compatibles para no provocar duplicados ni errores de claves foráneas.

### Paso 1: Hacer un respaldo antes de modificar la base II

No trabajes directamente sobre la única copia de la instalación original. Antes de transferir, genera un respaldo desde el motor de origen o mediante DBeaver.

En DBeaver, abre la conexión original, haz clic derecho sobre la base o esquema y selecciona **Tools > Backup** cuando esté disponible. Guarda el respaldo con un nombre que identifique motor y fecha, por ejemplo:

```text
backup_original_mysql_2026-09-17.sql
backup_original_postgres_2026-09-17.sql
```

Para SQL Server y Oracle, utiliza el mecanismo de respaldo configurado para esos motores o exporta el esquema y datos desde DBeaver. No elimines el respaldo hasta comprobar que la instalación II funciona.

### Paso 2: Crear o importar la estructura en la base II

1. Conéctate a la base II del mismo motor.
2. Ejecuta el archivo SQL que contiene `CREATE TABLE`, claves, índices, vistas y procedimientos.
3. Actualiza el panel de DBeaver con **Refresh**.
4. Comprueba que todas las tablas aparecen antes de copiar registros.

Si el archivo SQL fue creado para el mismo motor, normalmente puede ejecutarse sin cambios. Si el origen y destino son motores diferentes, revisa primero la sintaxis: los tipos `ENUM`, `BOOLEAN`, columnas `IDENTITY` o `AUTO_INCREMENT`, funciones de fecha, triggers y procedimientos no son idénticos entre MySQL, PostgreSQL, SQL Server y Oracle.

### Paso 3: Copiar los datos con DBeaver

Para mover una tabla o varias tablas entre conexiones:

1. En DBeaver, abre y conecta la base **original** y la base **II**.
2. En la conexión original, selecciona una o más tablas.
3. Haz clic derecho y elige **Export Data** o **Tools > Data Transfer**.
4. Como destino, selecciona **Database**.
5. Elige la conexión II del motor destino.
6. Selecciona o asigna la tabla de destino correspondiente.
7. Revisa el mapeo de columnas: nombre, tipo, nulos y valor por defecto.
8. Si las tablas II están vacías, usa el modo **Insert**. No selecciones opciones que borren o trunquen la tabla de origen.
9. Ejecuta la transferencia y revisa el registro final de DBeaver.

Cuando existen llaves foráneas, transfiere primero las tablas padre y después las dependientes. Por ejemplo, primero las tablas de catálogos o entidades principales y luego las tablas que contienen sus claves foráneas.

### Paso 4: Comprobar que la migración fue correcta

Compara el número de registros en la conexión original y en la conexión II. Para una tabla llamada `nombre_tabla`, ejecuta en ambas conexiones:

```sql
SELECT COUNT(*) AS total_registros
FROM nombre_tabla;
```

También revisa una muestra de datos. En MySQL y PostgreSQL usa:

```sql
SELECT *
FROM nombre_tabla
LIMIT 10;
```

En SQL Server usa:

```sql
SELECT TOP 10 *
FROM nombre_tabla;
```

En Oracle usa:

```sql
SELECT *
FROM nombre_tabla
FETCH FIRST 10 ROWS ONLY;
```

## 11.4 Reglas según el tipo de transferencia

| Caso | Recomendación |
| --- | --- |
| Original MySQL a MySQL II | Exporta estructura/datos o usa Data Transfer; los tipos son compatibles. |
| Original PostgreSQL a PostgreSQL II | Exporta estructura/datos o usa Data Transfer; los tipos son compatibles. |
| Original SQL Server a SQL Server II | Crea primero la base y estructura II; luego transfiere registros. |
| Original Oracle a Oracle II | Usa el mismo Service Name II y el esquema `admin`; luego importa estructura/datos. |
| Entre motores diferentes | Crea y revisa el esquema destino antes de mover datos; no copies SQL sin validar tipos y sintaxis. |

## 11.5 Evitar errores comunes

- No confundas los puertos de la original con los de la instancia II.
- No ejecutes una transferencia con el destino apuntando a la base original.
- No migres tablas dependientes antes de sus tablas padre si hay llaves foráneas.
- No copies contraseñas ni archivos `.env` en scripts SQL.
- No borres la instalación original hasta tener respaldo y validar los conteos de registros en la II.

## Resultado esperado

Podrás alternar entre la base original y la II desde DBeaver sin afectar datos. Además, cuando decidas migrar, tendrás un procedimiento controlado para copiar estructura y registros hacia la instancia II y verificar el resultado.
