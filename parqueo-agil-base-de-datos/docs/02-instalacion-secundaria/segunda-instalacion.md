# Guía completa: segunda instalación `parqueo_agil_control_de_estacionamientoII`

## Objetivo

Esta guía levanta una segunda instalación independiente de los cuatro motores del documento original:

1. MySQL 8.0
2. PostgreSQL 17
3. Microsoft SQL Server 2022
4. Oracle Database XE 21c

No reutiliza los contenedores, puertos, red ni directorios de datos de `parqueo_agil_control_de_estacionamiento`. Solo se crean los servidores y sus conexiones de DBeaver; no se incluye el código de creación de tablas.

> Ejecuta todos los comandos de esta guía en la terminal de **Ubuntu/WSL**. No los ejecutes en PowerShell ni en el editor SQL de DBeaver.

## Diferencias con la primera instalación

| Recurso | Primera instalación | Segunda instalación |
| --- | --- | --- |
| Nombre de base/PDB | `parqueo_agil_control_de_estacionamiento` | `parqueo_agil_control_de_estacionamientoII` |
| Carpeta raíz | `~/ia-lab` | `~/ia-lab-parqueo-agil-ii` |
| Red Docker | `ia-lab-network` | `parqueo-agil-ii-network` |
| MySQL | `mysql-server` / `3306` | `mysql-server-ii` / `3307` |
| PostgreSQL | `postgres-server` / `5432` | `postgres-server-ii` / `5433` |
| SQL Server | `mssql-server` / `1433` | `mssql-server-ii` / `1434` |
| Oracle XE | `oracle-server` / `1521` | `oracle-server-ii` / `1522` |

El uso de puertos distintos permite tener las dos instalaciones encendidas al mismo tiempo.

## 1. Preparación común

Verifica que Docker ya está listo, como en la primera guía:

```bash
sudo systemctl is-active docker
docker --version
docker compose version
sudo docker ps
```

Crea un directorio exclusivo para la instalación II, sus datos y sus respaldos:

```bash
mkdir -p ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/{mysql,postgres,mssql,oracle}
mkdir -p ~/ia-lab-parqueo-agil-ii/data/{mysql,postgres,mssql}
mkdir -p ~/ia-lab-parqueo-agil-ii/backups

tree ~/ia-lab-parqueo-agil-ii
```

La estructura será:

```text
~/ia-lab-parqueo-agil-ii/
├── backups/
├── data/
│   ├── mysql/
│   ├── mssql/
│   └── postgres/
└── services/
    └── motores-bd-ii/
        ├── mysql/
        ├── mssql/
        ├── oracle/
        └── postgres/
```

Crea la red que utilizarán exclusivamente estos cuatro contenedores:

```bash
sudo docker network inspect parqueo-agil-ii-network >/dev/null 2>&1 || \
sudo docker network create parqueo-agil-ii-network

sudo docker network ls | grep parqueo-agil-ii-network
```

En los ejemplos se utiliza `<TU_CONTRASENA_SEGURA>`. Sustitúyela por una clave propia antes del primer inicio y usa esa misma clave al conectar DBeaver. No incluyas los archivos `.env` en Git.

---

# 2. MySQL 8.0

## 2.1 Crear `docker-compose.yml`

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

## 2.2 Crear `.env`

```bash
cat > ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mysql/.env <<'EOF'
TZ=America/Bogota
MYSQL_ROOT_PASSWORD=<TU_CONTRASENA_SEGURA>
MYSQL_DATABASE=parqueo_agil_control_de_estacionamientoII
MYSQL_USER=admin
MYSQL_PASSWORD=<TU_CONTRASENA_SEGURA>
EOF
```

La imagen crea la base `parqueo_agil_control_de_estacionamientoII` y el usuario `admin` durante el primer arranque.

## 2.3 Validar y levantar el servicio

```bash
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mysql
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker logs mysql-server-ii --tail 30
```

Verifica la base desde el contenedor:

```bash
sudo docker exec -it mysql-server-ii mysql -u admin -p -e 'SHOW DATABASES;'
```

Al solicitarla, escribe el valor de `MYSQL_PASSWORD`.

## 2.4 Conectar desde DBeaver

Selecciona **Nueva conexión** > **MySQL** y completa:

| Campo | Valor |
| --- | --- |
| Host | `localhost` |
| Port | `3307` |
| Database | `parqueo_agil_control_de_estacionamientoII` |
| Username | `admin` |
| Password | valor de `MYSQL_PASSWORD` |

Pulsa **Test Connection** y luego **Finish**.

---

## 2.5 cambiar de una base de datos a otra 

# Configurar MySQL en el puerto 3306 para MySQL Workbench

## 1. Entrar a la carpeta del proyecto

```bash
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mysql
```

## 2. Verificar los archivos

```bash
ls -la
```

Debe aparecer el archivo:

```text
docker-compose.yml
```

## 3. Abrir el archivo Docker Compose

```bash
nano docker-compose.yml
```

Busca esta sección:

```yaml
ports:
  - "3307:3306"
```

Cámbiala por:

```yaml
ports:
  - "3306:3306"
```

### Guardar en Nano

Presiona:

```text
Ctrl + O
```

Luego:

```text
Enter
```

Y finalmente:

```text
Ctrl + X
```

---

## 4. Verificar que el puerto 3306 esté libre

Ejecuta:

```bash
sudo ss -ltnp | grep :3306
```

Si no aparece nada, el puerto está libre.

Si aparece otro servicio utilizando el puerto `3306`, primero hay que identificarlo antes de continuar.

---

## 5. Detener el contenedor actual

Desde la misma carpeta:

```bash
sudo docker compose down
```

> ⚠️ IMPORTANTE: No utilizar `docker compose down -v`, porque podría eliminar los volúmenes asociados a la base de datos.

---

## 6. Volver a levantar el servidor MySQL

```bash
sudo docker compose up -d
```

---

## 7. Verificar que el contenedor esté funcionando

```bash
sudo docker ps
```

El contenedor `mysql-server-ii` debería mostrar algo parecido a:

```text
0.0.0.0:3306->3306/tcp
```

También debe aparecer como:

```text
healthy
```

---

## 8. Verificar específicamente el puerto 3306

```bash
sudo ss -ltnp | grep :3306
```

Debe aparecer que Docker está escuchando en el puerto:

```text
3306
```

---

## 9. Probar la conexión a MySQL

Como el usuario `admin` ya fue comprobado y funciona:

```bash
sudo docker exec -it mysql-server-ii mysql -u admin -p
```

Cuando solicite:

```text
Enter password:
```

introducir:

```text
<TU_CONTRASENA_SEGURA>
```

Después debería aparecer:

```text
mysql>
```

---

## 10. Comprobar las bases de datos

Dentro de MySQL ejecutar:

```sql
SHOW DATABASES;
```

Debe aparecer:

```text
parqueo_agil_control_de_estacionamientoII
```

Para salir de MySQL:

```sql
exit;
```

---

# Configurar MySQL Workbench

Crear una conexión nueva en MySQL Workbench con:

```text
Connection Name: Parqueo Agil II
Hostname:        127.0.0.1
Port:            3306
Username:        admin
Password:        <TU_CONTRASENA_SEGURA>
```

La base de datos que quieres utilizar es:

```text
parqueo_agil_control_de_estacionamientoII
```

---

# Conexión

La conexión quedará de esta manera:

```text
MySQL Workbench
       |
       | 127.0.0.1:3306
       v
Ubuntu / Docker
       |
       | 3306:3306
       v
mysql-server-ii
       |
       v
parqueo_agil_control_de_estacionamientoII
```

---

# Comprobación final

Ejecutar:

```bash
sudo docker ps
```

Debe aparecer algo similar a:

```text
mysql-server-ii    mysql:8.0    Up ... (healthy)    0.0.0.0:3306->3306/tcp
```

Luego probar la conexión desde MySQL Workbench usando:

```text
Host:     127.0.0.1
Port:     3306
User:     admin
Password: <TU_CONTRASENA_SEGURA>
```

---

# IMPORTANTE

No ejecutar:

```bash
docker compose down -v
```

No eliminar:

```text
data/mysql
```

La carpeta `data/mysql` contiene los datos persistentes de MySQL.

La configuración final del puerto debe ser:

```yaml
ports:
  - "3306:3306"
```

# 3. PostgreSQL 17

## 3.1 Crear `docker-compose.yml`

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

## 3.2 Crear `.env`

```bash
cat > ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/postgres/.env <<'EOF'
TZ=America/Bogota
POSTGRES_USER=admin
POSTGRES_PASSWORD=<TU_CONTRASENA_SEGURA>
POSTGRES_DB=parqueo_agil_control_de_estacionamientoII
EOF
```

`POSTGRES_DB` y `POSTGRES_USER` se aplican solamente al inicializar un directorio de datos vacío. No cambies estos valores esperando que un servidor ya creado cambie automáticamente.

## 3.3 Validar y levantar el servicio

```bash
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/postgres
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker logs postgres-server-ii --tail 30
```

Verifica la conexión local:

```bash
sudo docker exec -it postgres-server-ii \
  psql -U admin -d parqueo_agil_control_de_estacionamientoII \
  -c 'SELECT current_database(), current_user;'
```

## 3.4 Conectar desde DBeaver

Selecciona **Nueva conexión** > **PostgreSQL** y completa:

| Campo | Valor |
| --- | --- |
| Host | `localhost` |
| Port | `5433` |
| Database | `parqueo_agil_control_de_estacionamientoII` |
| Username | `admin` |
| Password | valor de `POSTGRES_PASSWORD` |

Pulsa **Test Connection** y luego **Finish**.

---

# 4. Microsoft SQL Server 2022

## 4.1 Crear `docker-compose.yml`

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

## 4.2 Crear `.env`

```bash
cat > ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mssql/.env <<'EOF'
TZ=America/Bogota
ACCEPT_EULA=Y
MSSQL_SA_PASSWORD=<TU_CONTRASENA_SEGURA>
MSSQL_PID=Developer
EOF
```

`SA` es el usuario administrador. La clave debe cumplir complejidad: mayúscula, minúscula, número y símbolo. Defínela antes del primer inicio, ya que modificar el `.env` después no modifica el valor guardado en la instancia.

## 4.3 Validar y levantar el servicio

```bash
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/mssql
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker logs mssql-server-ii --tail 40
```

Espera hasta que el registro diga que SQL Server está listo para conexiones de cliente.

## 4.4 Crear la base II una sola vez

SQL Server no crea una base de usuario con el `.env`. Ejecuta este comando una vez, después de que el servidor esté listo:

```bash
sudo docker exec -it mssql-server-ii /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U SA -P '<TU_CONTRASENA_SEGURA>' -C -d master \
  -Q "IF DB_ID(N'parqueo_agil_control_de_estacionamientoII') IS NULL BEGIN EXEC(N'CREATE DATABASE [parqueo_agil_control_de_estacionamientoII]'); END"
```

Si cambiaste la contraseña del `.env`, usa la nueva clave también en el comando. Verifica la creación:

```bash
sudo docker exec -it mssql-server-ii /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U SA -P '<TU_CONTRASENA_SEGURA>' -C -d master \
  -Q "SELECT name FROM sys.databases WHERE name = N'parqueo_agil_control_de_estacionamientoII';"
```

La opción `-C` permite que `sqlcmd` acepte el certificado local del contenedor.

## 4.5 Conectar desde DBeaver

Selecciona **Nueva conexión** > **SQL Server** y completa:

| Campo | Valor |
| --- | --- |
| Host | `localhost` |
| Port | `1434` |
| Database | `parqueo_agil_control_de_estacionamientoII` |
| Username | `SA` |
| Password | valor de `MSSQL_SA_PASSWORD` |

Si DBeaver avisa de un certificado no confiable, activa **Trust server certificate** en SSL o Driver properties. Pulsa **Test Connection** y luego **Finish**.

---

# 5. Oracle Database XE 21c

## 5.1 Crear `docker-compose.yml`

Oracle usa un volumen Docker con nombre. Es independiente del volumen de la primera instalación y evita problemas de permisos de WSL sobre `/opt/oracle/oradata`.

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

No agregues `command:`: podría sustituir el arranque definido por la imagen de Oracle.

## 5.2 Crear `.env`

```bash
cat > ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/oracle/.env <<'EOF'
TZ=America/Bogota
ORACLE_PASSWORD=<TU_CONTRASENA_SEGURA>
ORACLE_DATABASE=parqueo_agil_control_de_estacionamientoII
EOF
```

`ORACLE_DATABASE` representa el PDB y su **Service Name**. No es un usuario; los usuarios iniciales son `SYSTEM` y `SYS`.

## 5.3 Validar y levantar el servicio

```bash
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/oracle
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker logs -f oracle-server-ii
```

La primera inicialización de Oracle puede tardar varios minutos. Cuando el registro indique que el servicio está listo, usa `Ctrl + C` para dejar de seguir los registros; el contenedor seguirá funcionando.

## 5.4 Crear el esquema del proyecto

La base/PDB y el esquema son conceptos distintos en Oracle. Crea el usuario propietario de las tablas una sola vez:

```bash
sudo docker exec -it oracle-server-ii bash
sqlplus 'system/<TU_CONTRASENA_SEGURA>@//localhost:1521/parqueo_agil_control_de_estacionamientoII'
```

Cuando veas `SQL>`, ejecuta:

```sql
CREATE USER admin IDENTIFIED BY "Admin_II_2026!" DEFAULT TABLESPACE USERS QUOTA UNLIMITED ON USERS;
GRANT CREATE SESSION, CREATE TABLE, CREATE SEQUENCE, CREATE VIEW, CREATE PROCEDURE, CREATE TRIGGER TO admin;
EXIT;
```

Después escribe `exit` para salir del contenedor. Si el usuario ya existe porque restauraste datos, no lo crees nuevamente.

## 5.5 Conectar desde DBeaver

Selecciona **Nueva conexión** > **Oracle**. En la pestaña **Basic**, completa:

| Campo | Valor |
| --- | --- |
| Host | `localhost` |
| Port | `1522` |
| Connection type | `Basic` |
| Service Name | `parqueo_agil_control_de_estacionamientoII` |
| Username | `admin` |
| Password | `Admin_II_2026!` o la clave que elegiste |
| Role | `Normal` |

Selecciona **Service Name**, no SID. Pulsa **Test Connection** y luego **Finish**.

---

# 6. Tablas existentes: punto importante

Los cuatro servidores de esta guía son totalmente nuevos. Por diseño, levantar un servidor independiente **no transfiere automáticamente** las tablas ni los registros de la primera instalación.

- Si ya cuentas con los archivos `.sql` de las tablas, conéctate desde DBeaver al motor correspondiente y ejecútalos allí.
- Si las tablas solo existen en los contenedores anteriores, expórtalas desde la primera instancia e impórtalas en la conexión II.
- Si ya restauraste un respaldo en el volumen/directorio nuevo, basta con verificarlas.

Usa esta consulta de verificación en DBeaver según el motor:

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

---

# 7. Operación y diagnóstico

## Consultar el estado y puertos

```bash
sudo docker ps --filter 'name=-server-ii'
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Debes ver los puertos `3307`, `5433`, `1434` y `1522`.

## Ver registros

```bash
sudo docker logs mysql-server-ii --tail 50
sudo docker logs postgres-server-ii --tail 50
sudo docker logs mssql-server-ii --tail 50
sudo docker logs oracle-server-ii --tail 50
```

## Detener un motor sin borrar su información

Ejemplo con PostgreSQL:

```bash
cd ~/ia-lab-parqueo-agil-ii/services/motores-bd-ii/postgres
sudo docker compose down
```

No ejecutes `docker compose down -v` si deseas conservar la información. `down` conserva los datos persistentes; el modificador `-v` sí elimina el volumen de Oracle y puede eliminar los volúmenes de un servicio.

## Conectar desde otro equipo

En el mismo computador, DBeaver debe usar `localhost`. Desde otro equipo de la red, utiliza la IP del anfitrión y abre únicamente el puerto necesario:

```bash
sudo ufw allow 3307/tcp
sudo ufw allow 5433/tcp
sudo ufw allow 1434/tcp
sudo ufw allow 1522/tcp
sudo ufw status
```

No abras esos puertos en una red pública.

## Resultado final

Al terminar tendrás cuatro conexiones nuevas en DBeaver, una por motor, dirigidas a `parqueo_agil_control_de_estacionamientoII`. Sus contenedores, puertos, red y datos quedan separados de `parqueo_agil_control_de_estacionamiento`.
