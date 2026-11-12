# Instalación y paso a paso de modelos de base de datos

> **Nota de seguridad:** las capturas de instalación se omitieron de la versión pública porque podían contener datos de configuración local. Todas las contraseñas fueron sustituidas por marcadores seguros. 

## Página web del docente

Escribo aquí la dirección de la página web del docente:

https://tecnogua.com/academic/site/bd/introduccion/

## instalamos los requerimientos 

comenzamos con la instalacion paso a paso de cada uno de los requerimientos 

- WSL2 instalado y funcionando
- Docker funcionando dentro de WSL
- Acceso a terminal bash en WSL

Instalar Docker y Compose


### Actualizar paquetes
``` bash
sudo apt update
```
evidencia



### Instalar dependencias
``` bash 
sudo apt install -y ca-certificates curl
```

evidencia



### Crear directorio para llaves
``` bash
sudo install -m 0755 -d /etc/apt/keyrings
```

evidencia



### Descargar llave GPG de Docker
``` bash 
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
``` 
evidencia



### Dar permisos
``` bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

evidencia


 
### Agregar repositorio
``` bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

evidencia



### Actualizar lista de paquetes
``` bash
sudo apt update
```
evidencia


### Instalar Docker
``` bash 
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

evidencia


``` bash 
sudo apt update
sudo apt install -y tree
```

evidencia


Verifica Docker:

``` bash
sudo docker --version
sudo docker compose version
```

evidencia


verifico si tengo instalado lo necesario usando los siguientes comandos en WSL.

```bash
sudo systemctl is-active docker
```
```bash
docker compose version
```
```bash
docker --version
```
 evidencia de que todo esta instalado bien



se agregan estos codigos para evitar erroes 

``` bash
sudo apt update
sudo apt upgrade -y
```
``` bash
sudo apt install --only-upgrade python3-software-properties software-properties-common
```
``` bash
sudo apt full-upgrade -y
```

# Instalo los requisitos para Docker

```bash
sudo apt-get install ca-certificates curl
```

## Agrego la llave oficial de Docker

### Add Docker's official GPG key:
```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
evidencia


### Agrego el repositorio de Docker

#### Add the repository to Apt sources:
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

evidencia



## Instalo Docker Compose plugin y compruebo la instalación

```bash
sudo systemctl stop unattended-upgrades
sudo apt install docker-compose-plugin
sudo docker --version
sudo docker compose version
```
evidencia 


# PRIMERA MAQUINA MSQL

##  Paso 1: Crear Carpetas

ejecuta para crear la carpeta:

``` bash
mkdir -p ~/ia-lab/services/motores-bd/{mysql,postgres,mssql,oracle}
mkdir -p ~/ia-lab/data/{mysql,postgres,mssql,oracle}
```

Verifica la estructura:

``` bash
cd ia-lab
tree
```

Debería verse así:

```         
~/ia-lab/
├── services/
│   └── motores-bd/
│       ├── mysql/
│       ├── postgres/
│       ├── mssql/
│       └── oracle/
└── data/
    ├── mysql/
    ├── postgres/
    ├── mssql/
    └── oracle/
```

evidencia


**Registro:**

despues de instalar los requeriminetos y docker compose plugin  creamos la primera maquina virtual . El resultado fue tal y como se esperaba en el diagrama, porque se crearon correctamente las carpetas `data` y `services`, junto con las carpetas de MySQL, PostgreSQL y MS SQL Server.

##  Paso 1.2: Crear la red Docker compartida

Creo una red Docker compartida para que todos los contenedores puedan comunicarse entre sí. Ejecuto este comando en mi terminal WSL:

```bash
docker network inspect ia-lab-network >/dev/null 2>&1 || docker network create ia-lab-network
```

en caso de un error se hace esto 

```bash
exit 
```
se entra a poweshell y se escribe esto 
```bash
wsl --shutdown
```
se entra a ubunto y se ejecuta esto 

```bash
groups
```

```bash
docker ps
```
y ya se soluciona 

evidencia 


Verifico que la red se haya creado correctamente:

```bash
docker network ls | grep ia-lab
```

Evidencia




Ejecuté los comandos en WSL. El resultado fue el esperado: se creó la red `ia-lab-network`, se mostró su identificador `6ebf13e6d30a` y apareció como una red `bridge` local al ejecutar la verificación.


# Instalación de Base de Datos

## paso 1.3 Crear el archivo `docker-compose.yml`

Creo el archivo de configuración de MySQL dentro de la carpeta correspondiente usando el siguiente comando:

```bash
cat > ~/ia-lab/services/motores-bd/mysql/docker-compose.yml << 'EOF'
services:
  mysql:
    image: mysql:8.0
    container_name: mysql-server
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "3306:3306"
    volumes:
      - ../../../data/mysql:/var/lib/mysql
      - /mnt/d/academia/bd:/backups
    command: >
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --bind-address=0.0.0.0
    networks:
      - ia-lab-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

networks:
  ia-lab-network:
    external: true
EOF
```

Intento habilitar UFW y permito el puerto `3306` de MySQL en el host:

```bash
sudo ufw allow 3306/tcp
sudo ufw enable
sudo ufw status
```

evidencia de que todo funciono bien 


## en caso de un error 

si ufw no está instalado. Instálalo primero:

``` bash
sudo apt install ufw -y
```
Luego ejecuta los comandos por separado:

```bash 
sudo ufw allow 3306/tcp
```

```bash 
sudo ufw enable
```

```bash 
sudo ufw status
```

### 1.4 Crear el archivo `.env`

Creo el archivo `.env` para configurar la zona horaria, la contraseña del usuario administrador y el nombre de la base de datos de MySQL:

```bash
cat > ~/ia-lab/services/motores-bd/mysql/.env << 'EOF'
TZ=America/Bogota
MYSQL_ROOT_PASSWORD=<TU_CONTRASENA_SEGURA>
MYSQL_DATABASE=parqueo_agil_control_de_estacionamiento
EOF



```

Evidencia



Creé correctamente el archivo `.env` con la zona horaria `America/Bogota`, la contraseña `<TU_CONTRASENA_SEGURA>` y la base de datos`parqueo_agil_control_de_estacionamiento`, tal como se observa en la evidencia`parqueo_agil_control_de_estacionamiento` es la base de datos del proyecto final del semestre.


### 1.5 Crear `README.md`

Creo el archivo `README.md` en la carpeta de MySQL:

```bash
touch ~/ia-lab/services/motores-bd/mysql/README.md
```

Después abro el archivo para agregar su contenido:

```bash
sudo nano ~/ia-lab/services/motores-bd/mysql/README.md
```

Dentro del archivo escribo lo siguiente:

````markdown
# MySQL 8.0 - Motor de Base de Datos

> **Acceso remoto habilitado.** Puerto expuesto en `0.0.0.0:3306`.
> **Usuario por defecto:** `root`
> **Base de datos inicial:** `parqueo_agil_control_de_estacionamiento`
> **Password:** `<TU_CONTRASENA_SEGURA>`

---

## Conectar desde WSL (local)

```bash
sudo docker exec -it mysql-server mysql -u root -p
# Password: <TU_CONTRASENA_SEGURA>
```
````

Después de guardar el contenido, muestro el archivo para comprobar que fue creado correctamente:

```bash
cat ~/ia-lab/services/motores-bd/mysql/README.md
```

Evidencia 


### 1.6 Levantar MySQL

Ingreso a la carpeta de MySQL y levanto el contenedor en segundo plano:

```bash
cd ~/ia-lab/services/motores-bd/mysql
sudo docker compose up -d
```

Verifico que el contenedor esté corriendo:

```bash
sudo docker ps | grep mysql-server
```

Después reviso los últimos 20 registros del contenedor:

```bash
sudo docker logs mysql-server --tail 20
```

evidencia 


### 1.7 Conectar localmente a MySQL

A diferencia de la guía del docente, donde se crea una base de datos de prueba (`bd_clase1`), aquí pasamos directo a la base de datos oficial del proyecto final: `parqueo_agil_control_de_estacionamiento`. No se crea un nuevo esquema porque la conexión ya está declarada en el .`env` mediante `MYSQL_DATABASE=parqueo_agil_control_de_estacionamiento`, así que simplemente la seleccionamos para trabajar.

La estructura de mi proyecto final es **ParqueoÁgil - Control de estacionamiento**. La referencia del proyecto es:


Me conecto localmente al motor de MySQL con el siguiente comando:

```bash
sudo docker exec -it mysql-server mysql -u root -p
```
evidencia 


Listo todas las bases de datos:

```sql
SHOW DATABASES;
```

Selecciono la base de datos `parqueo_agil_control_de_estacionamiento`, que corresponde al proyecto final del semestre:

```sql
USE parqueo_agil_control_de_estacionamiento;
```

Como la base de datos todavía no tiene las tablas del proyecto, las creo dentro de `parqueo_agil_control_de_estacionamiento`:

```sql
CREATE TABLE roles (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    nombre          VARCHAR(50) NOT NULL UNIQUE,
    descripcion     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE usuarios (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    rol_id          INT NOT NULL,
    nombre          VARCHAR(150) NOT NULL,
    username        VARCHAR(100) NOT NULL UNIQUE,
    email           VARCHAR(150) UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_usuarios_rol FOREIGN KEY (rol_id) REFERENCES roles(id)
) ENGINE=InnoDB;

CREATE TABLE clientes (
    id                  INT AUTO_INCREMENT PRIMARY KEY,
    tipo_documento      VARCHAR(20) NOT NULL,
    numero_documento    VARCHAR(30) NOT NULL,
    nombre              VARCHAR(150) NOT NULL,
    telefono            VARCHAR(30),
    email               VARCHAR(150),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT uq_clientes_documento UNIQUE (numero_documento)
) ENGINE=InnoDB;

CREATE TABLE vehiculos (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    cliente_id      INT NULL,
    placa           VARCHAR(15) NOT NULL,
    nombre          VARCHAR(150),
    descripcion     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT uq_vehiculos_placa UNIQUE (placa),
    CONSTRAINT fk_vehiculos_cliente FOREIGN KEY (cliente_id) REFERENCES clientes(id)
) ENGINE=InnoDB;

CREATE TABLE zonas (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    nombre          VARCHAR(100) NOT NULL,
    descripcion     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE espacios (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    zona_id         INT NOT NULL,
    nombre          VARCHAR(50) NOT NULL,
    descripcion     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_espacios_zona FOREIGN KEY (zona_id) REFERENCES zonas(id),
    CONSTRAINT uq_espacios_nombre_zona UNIQUE (zona_id, nombre)
) ENGINE=InnoDB;

CREATE TABLE tarifas (
    id                  INT AUTO_INCREMENT PRIMARY KEY,
    nombre              VARCHAR(100) NOT NULL,
    regla_calculo       ENUM('POR_HORA','POR_FRACCION','TARIFA_PLANA') NOT NULL,
    valor_base          DECIMAL(10,2) NOT NULL,
    vigencia_desde      DATE NOT NULL,
    vigencia_hasta      DATE NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE turnos_caja (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    usuario_id      INT NOT NULL,
    nombre          VARCHAR(100) NOT NULL,
    descripcion     VARCHAR(255),
    fecha_apertura  DATETIME NOT NULL,
    fecha_cierre    DATETIME NULL,
    monto_apertura  DECIMAL(10,2) NOT NULL DEFAULT 0,
    monto_cierre    DECIMAL(10,2) NULL,
    estado          ENUM('ABIERTO','CERRADO') NOT NULL DEFAULT 'ABIERTO',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_turnos_usuario FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
) ENGINE=InnoDB;

CREATE TABLE tickets (
    id                  INT AUTO_INCREMENT PRIMARY KEY,
    vehiculo_id         INT NOT NULL,
    espacio_id          INT NOT NULL,
    tarifa_id           INT NOT NULL,
    turno_caja_id       INT NOT NULL,
    placa               VARCHAR(15) NOT NULL,
    hora_ingreso        DATETIME NOT NULL,
    hora_salida         DATETIME NULL,
    evidencia           VARCHAR(255),
    estado              ENUM('ABIERTO','CERRADO') NOT NULL DEFAULT 'ABIERTO',
    monto_calculado     DECIMAL(10,2) NULL,
    recargo             DECIMAL(10,2) NOT NULL DEFAULT 0,
    perdida             BOOLEAN NOT NULL DEFAULT FALSE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_tickets_vehiculo FOREIGN KEY (vehiculo_id) REFERENCES vehiculos(id),
    CONSTRAINT fk_tickets_espacio FOREIGN KEY (espacio_id) REFERENCES espacios(id),
    CONSTRAINT fk_tickets_tarifa FOREIGN KEY (tarifa_id) REFERENCES tarifas(id),
    CONSTRAINT fk_tickets_turno FOREIGN KEY (turno_caja_id) REFERENCES turnos_caja(id)
) ENGINE=InnoDB;

CREATE INDEX idx_tickets_estado ON tickets(estado);
CREATE INDEX idx_tickets_placa ON tickets(placa);

CREATE TABLE abonos (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    cliente_id      INT NOT NULL,
    fecha           DATE NOT NULL,
    valor           DECIMAL(10,2) NOT NULL,
    estado          ENUM('ACTIVO','VENCIDO','CANCELADO') NOT NULL DEFAULT 'ACTIVO',
    observaciones   VARCHAR(255),
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_abonos_cliente FOREIGN KEY (cliente_id) REFERENCES clientes(id)
) ENGINE=InnoDB;

CREATE TABLE abono_vehiculo (
    abono_id        INT NOT NULL,
    vehiculo_id     INT NOT NULL,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (abono_id, vehiculo_id),
    CONSTRAINT fk_av_abono FOREIGN KEY (abono_id) REFERENCES abonos(id) ON DELETE CASCADE,
    CONSTRAINT fk_av_vehiculo FOREIGN KEY (vehiculo_id) REFERENCES vehiculos(id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE pagos (
    id                  INT AUTO_INCREMENT PRIMARY KEY,
    referencia_tipo     ENUM('TICKET','ABONO') NOT NULL,
    referencia_id       INT NOT NULL,
    usuario_id          INT NOT NULL,
    metodo              ENUM('EFECTIVO','TARJETA','TRANSFERENCIA','APP') NOT NULL,
    monto               DECIMAL(10,2) NOT NULL,
    fecha               DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    estado              ENUM('PENDIENTE','COMPLETADO','ANULADO') NOT NULL DEFAULT 'COMPLETADO',
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_pagos_usuario FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
) ENGINE=InnoDB;

CREATE INDEX idx_pagos_referencia ON pagos(referencia_tipo, referencia_id);

CREATE TABLE novedades (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    ticket_id       INT NOT NULL,
    usuario_id      INT NOT NULL,
    nombre          VARCHAR(100) NOT NULL,
    descripcion     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_novedades_ticket FOREIGN KEY (ticket_id) REFERENCES tickets(id),
    CONSTRAINT fk_novedades_usuario FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
) ENGINE=InnoDB;

DELIMITER $$
CREATE TRIGGER trg_tickets_no_doble_cierre
BEFORE UPDATE ON tickets
FOR EACH ROW
BEGIN
    IF OLD.estado = 'CERRADO'
       AND (NEW.hora_salida <> OLD.hora_salida OR NEW.estado <> OLD.estado)
    THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'No se puede modificar el cierre de un ticket ya cerrado (misma estancia).';
    END IF;
END$$
DELIMITER ;

CREATE VIEW vw_espacios_disponibles AS
SELECT e.id, e.nombre, e.zona_id
FROM espacios e
WHERE e.is_active = TRUE
  AND e.id NOT IN (
      SELECT t.espacio_id FROM tickets t WHERE t.estado = 'ABIERTO'
  );

INSERT INTO roles (nombre, descripcion) VALUES
    ('ADMIN', 'Administrador del sistema, acceso total'),
    ('OPERADOR', 'Gestiona ingreso y salida de vehiculos'),
    ('CAJA', 'Gestiona cobros y turnos de caja'),
    ('SUPERVISOR', 'Supervisa operacion y reportes'),
    ('AUDITOR', 'Solo lectura, auditoria de operaciones');

INSERT INTO usuarios (rol_id, nombre, username, email, password_hash) VALUES
    (1, 'Administrador', 'admin', 'admin@parqueoagil.local', SHA2('cambiar_esta_clave', 256));

INSERT INTO zonas (nombre, descripcion) VALUES
    ('Zona A', 'Parqueadero cubierto, primer piso'),
    ('Zona B', 'Parqueadero descubierto, patio exterior');

INSERT INTO espacios (zona_id, nombre, descripcion) VALUES
    (1, 'A-01', 'Espacio para automovil'),
    (1, 'A-02', 'Espacio para automovil'),
    (2, 'B-01', 'Espacio para motocicleta');

INSERT INTO tarifas (nombre, regla_calculo, valor_base, vigencia_desde) VALUES
    ('Tarifa hora carro', 'POR_HORA', 3000.00, CURDATE()),
    ('Tarifa fraccion moto', 'POR_FRACCION', 1000.00, CURDATE());
```

Después de crear las tablas, las listo para verificar que quedaron registradas:

```sql
SHOW TABLES;
```

Al consultar `parqueo_agil_control_de_estacionamiento`, verifico las tablas principales de mi proyecto:

```text
abono_vehiculo      
abonos    
clientes    
espacios   
novedades       
pagos       
roles     
tarifas     
tickets 
turnos_caja    
usuarios   
vehiculos
vw_espacios_disponibles               
zonas                                 
```


Listo los campos de una tabla:

```sql
DESCRIBE nombre_tabla;
```

También puedo listar los campos con:

```sql
SHOW COLUMNS FROM nombre_tabla;
```

Salgo del motor de MySQL:

```sql
EXIT;
```

evidencia 




Registro:

Me conecté localmente al motor de MySQL, seleccioné la base de datos del proyecto, creé y configuré las tablas correspondientes al sistema de gestión de parqueadero: roles, usuarios, clientes, vehiculos, zonas, espacios, tarifas, turnos_caja, tickets, abonos, abono_vehiculo, pagos y novedades. También se establecieron las claves primarias, claves foráneas, restricciones, índices, el trigger para evitar modificaciones en tickets cerrados y la vista para consultar los espacios disponibles. Finalmente, verifiqué la creación de las tablas mediante SHOW TABLES y salí correctamente del motor de MySQL.

### 1.8 Crear un usuario propio con acceso remoto

Primero me conecto como root al motor de MySQL que se encuentra ejecutándose en Docker:

``` bash 
sudo docker exec -it mysql-server mysql -u root -p
```

Después de ingresar la contraseña de root, ejecuto las instrucciones SQL correspondientes a mi proyecto de gestión de parqueadero.

Parte explicativa:

Código original de la guía del docente:

```sql
-- Crear la base de datos
CREATE DATABASE parqueo_agil_control_de_estacionamiento CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Crear usuario propio con acceso desde cualquier equipo (%)
CREATE USER 'admin'@'%' IDENTIFIED BY '<TU_CONTRASENA_SEGURA>';

-- Dar permisos sobre la base de datos
GRANT ALL PRIVILEGES ON parqueo_agil_control_de_estacionamiento.* TO 'admin'@'%';
FLUSH PRIVILEGES;
```
Código adaptado a mi proyecto:

En mi proyecto, la base de datos se denomina `parqueo_agil_control_de_estacionamiento`. Para administrar el sistema se utiliza el usuario `admin,` con acceso remoto desde cualquier equipo mediante `%`, y la contraseña establecida es `<TU_CONTRASENA_SEGURA>`.

``` sql

-- Crear la base de datos del proyecto
CREATE DATABASE IF NOT EXISTS parqueo_agil_control_de_estacionamiento
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;

-- Seleccionar la base de datos del proyecto
USE parqueo_agil_control_de_estacionamiento;

-- Crear usuario administrador con acceso remoto
CREATE USER 'admin'@'%' IDENTIFIED BY '<TU_CONTRASENA_SEGURA>';

-- Otorgar todos los privilegios sobre la base de datos parqueo_agil_control_de_estacionamiento 
GRANT ALL PRIVILEGES ON parqueo_agil_control_de_estacionamiento;.* TO 'admin'@'%';
FLUSH PRIVILEGES;
```
evidenvia



**Aplicado en mi proyecto:**

Creo el usuario `admin` con la contraseña `<TU_CONTRASENA_SEGURA>` y le otorgo permisos para conectarse remotamente a la base de datos `parqueo_agil_control_de_estacionamiento`:

```sql

CREATE USER 'admin'@'%' IDENTIFIED BY '<TU_CONTRASENA_SEGURA>';
GRANT ALL PRIVILEGES ON parqueo_agil_control_de_estacionamiento
.* TO 'admin'@'%';
FLUSH PRIVILEGES;
```

evidencia 


**nota:**

muestra el error en admin por que esta creado y se corrigue en el comando que esta abajo 


### 1.9 Conectar remotamente desde cualquier equipo con DBeaver

Para conectarme de forma remota a MySQL utilizo DBeaver. Necesito los siguientes datos:

- **Motor:** MySQL.
- **Servidor:** la dirección IP del equipo donde está ejecutándose Docker.
- **Puerto:** `3306`.
- **Base de datos:** `parqueo_agil_control_de_estacionamiento`.
- **Usuario:** `admin`.
- **Contraseña:** `<TU_CONTRASENA_SEGURA>`.

Para encontrar y comprobar estos datos, ejecuto los siguientes comandos en WSL.

Busco la dirección IP del equipo donde se ejecuta Docker:

```bash
ip a
```

En el resultado busco la interfaz `eth0` y tomo la dirección que aparece después de `inet`. En mi caso, la dirección es `172.18.50.255`.

Compruebo que el contenedor MySQL esté ejecutándose y que el puerto `3306` esté publicado:

```bash
sudo docker ps --filter "name=mysql-server" --format "table {{.Names}}\t{{.Ports}}"
```

Compruebo que el servidor MySQL esté funcionando y responda correctamente:

```bash
sudo docker exec mysql-server mysqladmin ping -h localhost -u root -p<TU_CONTRASENA_SEGURA>
```

El resultado fue `mysqld is alive`, por lo que confirmé que el servidor MySQL está funcionando correctamente. También apareció una advertencia indicando que usar la contraseña directamente en la línea de comandos puede ser inseguro.

También compruebo que el puerto esté permitido por UFW:

```bash
sudo ufw status
```

Consulto el nombre de la base de datos configurada en el archivo `.env`:

```bash
grep '^MYSQL_DATABASE=' ~/ia-lab/services/motores-bd/mysql/.env
```

Consulto los usuarios de MySQL y sus equipos de acceso:

```bash
sudo docker exec -it mysql-server mysql -u root -p -e "SELECT User, Host FROM mysql.user;"
```

Compruebo que el usuario `admin` tenga permisos sobre `parqueo_agil_control_de_estacionamiento`:

```bash
sudo docker exec -it mysql-server mysql -u root -p -e "SHOW GRANTS FOR 'admin'@'%';"
```

```bash
sudo docker exec -it mysql-server mysql -u root -p -e "ALTER USER 'admin'@'%' IDENTIFIED WITH mysql_native_password BY '<TU_CONTRASENA_SEGURA>'; FLUSH PRIVILEGES;"
```

```bash
sudo docker exec -it mysql-server mysql -u root -p -e "SELECT User, Host, plugin FROM mysql.user WHERE User = 'admin';"
```

evidencia



En DBeaver realizo los siguientes pasos:

1. Abro DBeaver.
2. Selecciono **Nueva conexión**.
3. Elijo el controlador **MySQL**.
4. Busco la interfaz `eth0` en el resultado de `ip a` y escribo ` 172.17.55.39` en el campo **Host**.
5. Escribo `3306` en **Port**.
6. Escribo `admin` en **Username**.
7. Escribo `<TU_CONTRASENA_SEGURA>` en **Password**.
8. Escribo `parqueo_agil_control_de_estacionamiento` en **Database**.
9. Presiono **Test Connection**.
10. Si la prueba es correcta, presiono **Finish** para guardar la conexión.


evidencia de la aplicacion DB







**Registro:**

Después de corregir el método de autenticación del usuario `admin`, probé nuevamente la conexión en DBeaver. El resultado fue exitoso: apareció el mensaje `Conectado (244 ms)` y se confirmó la conexión con MySQL `8.0.46` usando la base de datos `parqueo_agil_control_de_estacionamiento`.

### 1.10 Backup de la base de datos

Realizo un backup de la base de datos `parqueo_agil_control_de_estacionamiento` y guardo el archivo en la carpeta de respaldos:

```bash
sudo docker exec mysql-server mysqldump -u root -p<TU_CONTRASENA_SEGURA> parqueo_agil_control_de_estacionamiento > /mnt/d/academia/bd/backup_parqueo_agil_control_de_estacionamiento_$(date +%Y%m%d).sql
```

Verifico que el archivo de backup se haya creado correctamente:

```bash
ls -lh /mnt/d/academia/bd/backup_parqueo_agil_control_de_estacionamiento_$(date +%Y%m%d).sql
```

**Evidencia (imagen):**



**Registro:**

Al ejecutar el backup, apareció el error `Permission denied` porque no tenía permisos para crear el archivo directamente en `/mnt/d/academia/bd/` mediante la redirección `>`. El backup todavía no se creó.

**Diferencia entre la guía del docente y mi solución:**

El comando original de la página web del docente es:

```bash
sudo docker exec mysql-server mysqldump -u root -pMiNiCo57** tecnogua > /mnt/d/academia/bd/backup_tecnogua_$(date +%Y%m%d).sql
```

En ese comando, `mysqldump` se ejecuta dentro del contenedor, pero la redirección `>` intenta crear el archivo desde WSL en `/mnt/d/academia/bd/`. En mi caso, esa redirección produjo el error `Permission denied`.

Mi solución fue crear el backup directamente dentro del contenedor, usando la carpeta `/backups`, que está conectada con `/mnt/d/academia/bd/`. Además, cambié los datos de la guía por los de mi proyecto: la base de datos es `parqueo_agil_control_de_estacionamiento` y la contraseña de `root` es `<TU_CONTRASENA_SEGURA>`.

Como la carpeta `/mnt/d/academia/bd/` está montada en el contenedor como `/backups`, genero el archivo directamente dentro del contenedor. Así no uso la redirección de Bash en WSL ni `tee`:

```bash
sudo docker exec -it mysql-server mysql -u root -p
```

Dentro de MySQL creo el backup directamente en la carpeta montada:

```sql
SYSTEM mysqldump -u root -p<TU_CONTRASENA_SEGURA> parqueo_agil_control_de_estacionamiento > /backups/backup_parqueo_agil_control_de_estacionamiento_$(date +%Y%m%d).sql;
```

Salgo del motor:

```sql
EXIT;
```

Después verifico que el backup se haya creado correctamente:

```bash
ls -lh /mnt/d/academia/bd/backup_parqueo_agil_control_de_estacionamiento_$(date +%Y%m%d).sql
```

**Evidencia (imagen):**



**Registro:**

Creé el backup de la base de datos `parqueo_agil_control_de_estacionamiento` directamente dentro del contenedor y verifiqué desde WSL que el archivo se generó correctamente en `/mnt/d/academia/bd/`.

### 1.10 Variables clave del `.env`

Estas son las variables principales que configuré en el archivo `.env`:

| Variable | Descripción |
| --- | --- |
| `TZ` | Zona horaria: `America/Bogota` |
| `MYSQL_ROOT_PASSWORD` | Contraseña del usuario root: `<TU_CONTRASENA_SEGURA>` |
| `MYSQL_DATABASE` | Base de datos inicial: `parqueo_agil_control_de_estacionamiento` |



**Registro:**

Registré las variables principales del archivo `.env`: la zona horaria, la contraseña del usuario root y la base de datos inicial del proyecto final, `parqueo_agil_control_de_estacionamiento`.

Con este registro finalizo la instalación del motor MySQL para mi proyecto `parqueo_agil_control_de_estacionamiento`. Dejé el contenedor funcionando, configuré la base de datos, creé sus tablas, habilité el acceso remoto, comprobé la conexión desde DBeaver y generé el backup correspondiente.

## 2. PostgreSQL

Ahora continúo con la instalación y configuración del motor PostgreSQL siguiendo el mismo procedimiento de registro paso a paso.

### 2.1 Crear `docker-compose.yml`

Creo el archivo de configuración de PostgreSQL en la carpeta correspondiente:

```bash
cat > ~/ia-lab/services/motores-bd/postgres/docker-compose.yml << 'EOF'
services:
  postgres:
    image: postgres:17
    container_name: postgres-server
    restart: unless-stopped

    env_file:
      - .env

    ports:
      - "5432:5432"

    volumes:
      # Datos persistentes dentro de WSL
      - ../../../data/postgres:/var/lib/postgresql/data

      # Backups accesibles desde Windows (D:)
      - /mnt/d/academia/bd:/backups

    command:
      - postgres
      - -c
      - "listen_addresses=*"

    networks:
      - ia-lab-network

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

networks:
  ia-lab-network:
    external: true
EOF
```

Habilito UFW y permito el puerto `5432` de PostgreSQL en el host:

```bash
sudo ufw allow 5432/tcp
sudo ufw enable
sudo ufw status
```

**Evidencia (imagen):**





**Registro:**

Creé el archivo `docker-compose.yml` de PostgreSQL con la imagen `postgres:17`, el puerto `5432`, el almacenamiento persistente, la carpeta de backups y la red compartida `ia-lab-network`. También permití el puerto `5432` en UFW.

### 2.2 Crear el archivo `.env`

Creo el archivo `.env` para configurar PostgreSQL con los datos de mi proyecto `parqueo_agil_control_de_estacionamiento`:

```bash
cat > ~/ia-lab/services/motores-bd/postgres/.env << 'EOF'
TZ=America/Bogota
POSTGRES_USER=postgres
POSTGRES_PASSWORD=<TU_CONTRASENA_SEGURA>
POSTGRES_DB=parqueo_agil_control_de_estacionamiento
EOF
```

Para comprobar que el archivo `.env` quedó creado con los datos correctos, muestro su contenido:

```bash
cat ~/ia-lab/services/motores-bd/postgres/.env
```

Debo verificar que aparezcan estas variables:

```text
TZ=America/Bogota
POSTGRES_USER=postgres
POSTGRES_PASSWORD=<TU_CONTRASENA_SEGURA>
POSTGRES_DB=parqueo_agil_control_de_estacionamiento
```

**Evidencia (imagen):**



**Registro:**


**Registro:**

Creé el archivo `.env` de PostgreSQL con el usuario `postgres`, la contraseña `<TU_CONTRASENA_SEGURA>` y la base de datos `parqueo_agil_control_de_estacionamiento`.

### 2.2.1 Si se quiere cambiar la contraseña del `.env`

Para editar la clave del archivo `.env` en clase, ingreso a la carpeta de PostgreSQL y abro el archivo:

```bash
cd ~/ia-lab/services/motores-bd/postgres
sudo nano .env
```

Cambio la clave por:

```env
POSTGRES_PASSWORD=<TU_CONTRASENA_SEGURA>
```

Guardo los cambios con:

```text
Ctrl + O  ==>  para guardar
```

Después presiono Enter para confirmar el nombre del archivo y salgo del editor con:

```text
Ctrl + X  ==>  para salir
```

### 2.3 Crear `README.md`

Creo el archivo `README.md` en la carpeta de PostgreSQL:

```bash
touch ~/ia-lab/services/motores-bd/postgres/README.md
```

Después abro el archivo para agregar su contenido:

```bash
sudo nano ~/ia-lab/services/motores-bd/postgres/README.md
```

Dentro del archivo escribo lo siguiente:

````markdown
# PostgreSQL 17 - Motor de Base de Datos

> **Acceso remoto habilitado.** Puerto expuesto en `0.0.0.0:5432`.
> **Usuario por defecto:** `postgres`
> **Base de datos inicial:** `parqueo_agil_control_de_estacionamiento`
> **Password:** `<TU_CONTRASENA_SEGURA>`

---

## Conectar desde WSL (local)

```bash
sudo docker exec -it postgres-server psql -U postgres -d parqueo_agil_control_de_estacionamiento
# Password: <TU_CONTRASENA_SEGURA>
```
````

Después de guardar el contenido, muestro el archivo para comprobar que fue creado correctamente:

```bash
cat ~/ia-lab/services/motores-bd/postgres/README.md
```

**Evidencia (imagen):**



**Registro:**


**Registro:**

Creé el archivo `README.md` de PostgreSQL y comprobé su contenido con el comando `cat`. Registré el puerto `5432`, el usuario `postgres`, la base de datos `parqueo_agil_control_de_estacionamiento` y el comando para conectarme desde WSL.

### 2.4 Levantar PostgreSQL

Ingreso a la carpeta de PostgreSQL y levanto el contenedor en segundo plano:

```bash
cd ~/ia-lab/services/motores-bd/postgres
```

```bash
sudo docker compose up -d
```

Verifico que el contenedor esté corriendo:

```bash
sudo docker ps | grep postgres-server
```

Después reviso los últimos 20 registros del contenedor:

```bash
sudo docker logs postgres-server --tail 20
```

**Evidencia (imagen):**


**Registro:**

Al ejecutar `sudo docker compose up -d`, PostgreSQL no se levantó porque apareció el error `mapping key "networks" already defined at line 25` en la línea 35 del archivo YAML. El contenedor `postgres-server` todavía no pudo iniciarse.

### 2.4.1 Revisar y solucionar el error del archivo YAML

#### Código original

La parte original del archivo tenía la configuración de `networks` indentada con tabulaciones:

```yaml
services:
  postgres:
    networks:
      - ia-lab-network

networks:
  ia-lab-network:
    external: true
```

#### Error encontrado

El archivo produjo el error `mapping key "networks" already defined` porque YAML no permite tabulaciones para la indentación y las claves `networks` terminaron siendo interpretadas en el mismo nivel. La primera clave pertenece al servicio `postgres` y la segunda debe pertenecer al nivel principal del archivo.

Entro al archivo para reemplazar manualmente el código original:

```bash
cd ~/ia-lab/services/motores-bd/postgres
sudo nano docker-compose.yml
```

#### Código corregido

Después de entrar al archivo, reemplazo el contenido por el siguiente YAML, usando espacios:

```yaml
services:
  postgres:
    image: postgres:17
    container_name: postgres-server
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "5432:5432"
    volumes:
      - ../../../data/postgres:/var/lib/postgresql/data
      - /mnt/d/academia/bd:/backups
    command:
      - postgres
      - -c
      - "listen_addresses=*"
    networks:
      - ia-lab-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

networks:
  ia-lab-network:
    external: true
```

Guardo con `Ctrl + O`, presiono Enter para confirmar y salgo con `Ctrl + X`.

Después valido el archivo corregido:

```bash
sudo docker compose config
```

Si no aparecen errores, vuelvo a levantar PostgreSQL y verifico el contenedor:

```bash
sudo docker compose up -d
sudo docker ps | grep postgres-server
sudo docker logs postgres-server --tail 20
```

**Evidencia (imagen):**



**Registro:**

Corregí la estructura del archivo `docker-compose.yml`, validé la configuración y levanté PostgreSQL correctamente. El contenedor `postgres-server` quedó funcionando y los registros confirmaron su inicio.

### 2.5 Conectar localmente a PostgreSQL

Este paso es diferente al de la guía del docente porque trabajo directamente con `parqueo_agil_control_de_estacionamiento`, la base de datos de mi proyecto final, en lugar de crear otra base de datos de práctica.

La estructura de mi proyecto final es **parqueo_agil_control_de_estacionamiento - Bicicletas compartidas**:


Me conecto localmente al motor de PostgreSQL:

```bash
sudo docker exec -it postgres-server psql -U postgres -d parqueo_agil_control_de_estacionamiento
```

Listo todas las bases de datos:

```sql
\l
```

Selecciono la base de datos del proyecto:

```sql
\c parqueo_agil_control_de_estacionamiento
```

Como PostgreSQL todavía no tiene las tablas del proyecto, las creo dentro de `parqueo_agil_control_de_estacionamiento`:

```sql
CREATE TABLE cliente (
  id SERIAL PRIMARY KEY,
  tipo_documento VARCHAR(30) NOT NULL,
  numero_documento VARCHAR(50) NOT NULL UNIQUE,
  nombre VARCHAR(100) NOT NULL,
  telefono VARCHAR(30),
  email VARCHAR(150),
  is_active BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE estacion (
  id SERIAL PRIMARY KEY,
  nombre VARCHAR(100) NOT NULL,
  descripcion VARCHAR(255),
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE bicicleta (
  id SERIAL PRIMARY KEY,
  nombre VARCHAR(100) NOT NULL,
  descripcion VARCHAR(255),
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE anclaje (
  id SERIAL PRIMARY KEY,
  estacion_id INT NOT NULL REFERENCES estacion(id),
  bicicleta_id INT REFERENCES bicicleta(id),
  nombre VARCHAR(100) NOT NULL,
  descripcion VARCHAR(255),
  is_active BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE tarifa (
  id SERIAL PRIMARY KEY,
  nombre VARCHAR(100) NOT NULL,
  regla_calculo VARCHAR(255),
  valor_base NUMERIC(10, 2) NOT NULL,
  vigencia_desde DATE,
  vigencia_hasta DATE,
  is_active BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE reserva (
  id SERIAL PRIMARY KEY,
  cliente_id INT NOT NULL REFERENCES cliente(id),
  fecha_inicio TIMESTAMP NOT NULL,
  fecha_fin TIMESTAMP,
  estado VARCHAR(30) NOT NULL,
  observaciones TEXT
);

CREATE TABLE alquiler (
  id SERIAL PRIMARY KEY,
  reserva_id INT REFERENCES reserva(id),
  bicicleta_id INT NOT NULL REFERENCES bicicleta(id),
  cliente_id INT NOT NULL REFERENCES cliente(id),
  fecha_inicio TIMESTAMP NOT NULL,
  fecha_fin TIMESTAMP,
  total NUMERIC(10, 2),
  estado VARCHAR(30) NOT NULL,
  observaciones TEXT
);

CREATE TABLE evento_alquiler (
  id SERIAL PRIMARY KEY,
  alquiler_id INT NOT NULL REFERENCES alquiler(id),
  tipo VARCHAR(50) NOT NULL,
  fecha TIMESTAMP NOT NULL,
  cantidad INT,
  observaciones TEXT,
  estado VARCHAR(30) NOT NULL
);

CREATE TABLE penalidad (
  id SERIAL PRIMARY KEY,
  alquiler_id INT NOT NULL REFERENCES alquiler(id),
  tipo VARCHAR(100) NOT NULL,
  fecha DATE NOT NULL,
  valor NUMERIC(10, 2) NOT NULL,
  estado VARCHAR(30) NOT NULL,
  observaciones TEXT
);

CREATE TABLE mantenimiento (
  id SERIAL PRIMARY KEY,
  bicicleta_id INT NOT NULL REFERENCES bicicleta(id),
  tipo VARCHAR(100) NOT NULL,
  fecha_programada DATE,
  fecha_cierre DATE,
  costo NUMERIC(10, 2),
  estado VARCHAR(30) NOT NULL
);

CREATE TABLE pago (
  id SERIAL PRIMARY KEY,
  referencia_tipo VARCHAR(50) NOT NULL,
  referencia_id INT NOT NULL,
  metodo VARCHAR(50) NOT NULL,
  monto NUMERIC(10, 2) NOT NULL,
  fecha TIMESTAMP NOT NULL,
  estado VARCHAR(30) NOT NULL
);
```

**Evidencia (imagen):**


**Registro:**

Me conecté localmente a PostgreSQL, seleccioné `parqueo_agil_control_de_estacionamiento` y creé las tablas del proyecto final correctamente.

### 2.5.1 Consultar, editar o eliminar información

Para comprobar qué tablas existen dentro de `parqueo_agil_control_de_estacionamiento`, ejecuto:

```sql
\dt
```

Consulto los campos de una tabla:

```sql
\d bicicleta
```

También puedo cambiar de base de datos con:

```sql
\c parqueo_agil_control_de_estacionamiento
```

Para eliminar una base de datos, primero cambio a `postgres` y luego ejecuto:

```sql
\c postgres
DROP DATABASE parqueo_agil_control_de_estacionamiento;
```

Salgo del motor:

```sql
\q
```
### 2.5.2 Crear un usuario propio con acceso remoto

Me conecto primero como `postgres` al motor de PostgreSQL:

```bash
sudo docker exec -it postgres-server psql -U postgres -d parqueo_agil_control_de_estacionamiento
```

Después de ingresar la contraseña de `postgres`, ejecuto las instrucciones SQL.

**Código original de la guía del docente:**

Reemplaza `usuario`, `password` y `nombre_bd` por los valores de tu proyecto:

```sql
CREATE DATABASE nombre_bd;
CREATE USER usuario WITH PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE nombre_bd TO usuario;
ALTER DATABASE nombre_bd OWNER TO usuario;
```

**Código que colocaré en mi caso:**

La base de datos utilizada es `parqueo_agil_control_de_estacionamiento`, el usuario propio es `admin` y la contraseña es `<TU_CONTRASENA_SEGURA>`. Como la base de datos ya fue creada, solo creo el usuario y le asigno permisos de superusuario para permitir el acceso remoto y la administración del proyecto:

```sql
CREATE USER admin WITH PASSWORD '<TU_CONTRASENA_SEGURA>';
ALTER USER admin WITH SUPERUSER;
```

También puedo verificar que el usuario fue creado correctamente con:

```sql
\du
```

**Evidencia (imagen):**


**Registro:**

Me conecté como `postgres` y creé el usuario propio `admin` con la contraseña `<TU_CONTRASENA_SEGURA>` para trabajar con la base de datos `parqueo_agil_control_de_estacionamiento` y acceder al motor PostgreSQL de forma remota.

### 2.6 Conectar remotamente desde cualquier equipo

Para conectarme de forma remota a PostgreSQL utilizo DBeaver. Necesito los siguientes datos:

- **Motor:** PostgreSQL.
- **Servidor:** la dirección IP del equipo donde está ejecutándose Docker.
- **Puerto:** `5432`.
- **Base de datos:** `parqueo_agil_control_de_estacionamiento`.
- **Usuario:** `admin`.
- **Contraseña:** `<TU_CONTRASENA_SEGURA>`.

Para encontrar y comprobar estos datos, ejecuto los siguientes comandos en WSL.

Busco la dirección IP del equipo donde se ejecuta Docker:

```bash
ip a
```

En el resultado busco la interfaz `eth0` y tomo la dirección que aparece después de `inet`.

Compruebo que el contenedor PostgreSQL esté ejecutándose y que el puerto `5432` esté publicado:

```bash
sudo docker ps --filter "name=postgres-server" --format "table {{.Names}}\t{{.Ports}}"
```

Compruebo que el servidor PostgreSQL esté funcionando y acepte conexiones:

```bash
sudo docker exec postgres-server pg_isready -U postgres -d parqueo_agil_control_de_estacionamiento
```

También compruebo que el puerto esté permitido por UFW:

```bash
sudo ufw status
```

**Evidencia (imagen):**


**Registro:**

Comprobé que la interfaz `eth0` tiene la dirección IP `172.18.50.255`. También confirmé que el contenedor `postgres-server` está activo y publica el puerto `5432` mediante `0.0.0.0:5432->5432/tcp`. El comando `pg_isready` mostró `/var/run/postgresql:5432 - accepting connections`, por lo que PostgreSQL está funcionando correctamente. Además, UFW está activo y permite el puerto `5432` para conexiones IPv4 e IPv6.

Con estos resultados, en DBeaver debo seleccionar el controlador **PostgreSQL** y utilizar los siguientes datos:

- **Host:** `172.18.50.255`
- **Port:** `5432`
- **Database:** `parqueo_agil_control_de_estacionamiento`
- **Username:** `admin`
- **Password:** `<TU_CONTRASENA_SEGURA>`

### 2.6.1 Hacerlo desde DBeaver

En DBeaver realizo los siguientes pasos:

1. Abro DBeaver.
2. Selecciono **Nueva conexión**.
3. Elijo el controlador **PostgreSQL**.
4. Escribo `172.18.50.255` en el campo **Host**.
5. Escribo `5432` en **Port**.
6. Escribo `admin` en **Username**.
7. Escribo `<TU_CONTRASENA_SEGURA>` en **Password**.
8. Escribo `parqueo_agil_control_de_estacionamiento` en **Database**.
9. Presiono **Test Connection**.
10. Si la prueba es correcta, presiono **Finish** para guardar la conexión.

**Evidencia (imagen):**



**Registro:**

Configuré y probé en DBeaver la conexión remota a PostgreSQL utilizando el host `172.18.50.255`, el puerto `5432`, la base de datos `parqueo_agil_control_de_estacionamiento` y el usuario `admin` con la contraseña `<TU_CONTRASENA_SEGURA>`.

### 2.7 Backup de una base de datos

**Código original de la guía del docente:**

El siguiente comando pertenece únicamente al ejemplo de la guía del docente. No utilizo la base de datos `tecnogua`; en mi proyecto trabajo con `parqueo_agil_control_de_estacionamiento`.

```bash
sudo docker exec postgres-server pg_dump -U postgres -d tecnogua > /mnt/d/academia/bd/backup_tecnogua_$(date +%Y%m%d).sql
```

**Código que colocaré en mi caso:**

En mi proyecto utilizo la base de datos `parqueo_agil_control_de_estacionamiento`, la contraseña `<TU_CONTRASENA_SEGURA>` y el nombre de archivo `backup_parqueo_agil_control_de_estacionamiento`. Para evitar el error de permisos de la redirección de WSL, genero el archivo directamente dentro del contenedor. La carpeta `/backups` está conectada con `/mnt/d/academia/bd/`:

```bash
sudo docker exec postgres-server pg_dump -U postgres -d parqueo_agil_control_de_estacionamiento -f /backups/backup_parqueo_agil_control_de_estacionamiento_$(date +%Y%m%d).sql
```

Compruebo el archivo generado desde WSL:

```bash
ls -lh /mnt/d/academia/bd/backup_parqueo_agil_control_de_estacionamiento_$(date +%Y%m%d).sql
```

Para ver lo que está guardado en el archivo `.env` de PostgreSQL, ejecuto:

```bash
cat ~/ia-lab/services/motores-bd/postgres/.env
```

El contenido esperado es:

```env
TZ=America/Bogota
POSTGRES_USER=postgres
POSTGRES_PASSWORD=<TU_CONTRASENA_SEGURA>
POSTGRES_DB=parqueo_agil_control_de_estacionamiento
```

**Evidencia (imagen):**



**Registro:**

Realicé el backup únicamente de la base de datos `parqueo_agil_control_de_estacionamiento` y verifiqué el archivo generado en `/mnt/d/academia/bd/`. También mostré el contenido del archivo `.env`, donde están guardados el usuario `postgres`, la contraseña `<TU_CONTRASENA_SEGURA>` y la base de datos `parqueo_agil_control_de_estacionamiento`.

## 3. MS SQL Server

### 3.1 Crear el archivo `docker-compose.yml`

Creo el archivo `docker-compose.yml` para MS SQL Server:

```bash
cat > ~/ia-lab/services/motores-bd/mssql/docker-compose.yml << 'EOF'
services:
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: mssql-server
    restart: unless-stopped
    user: root
    env_file:
      - .env
    ports:
      - "0.0.0.0:1433:1433"
    volumes:
      - ../../../data/mssql:/var/opt/mssql
      - /mnt/d/academia/bd:/backups
    networks:
      - ia-lab-network
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P $$MSSQL_SA_PASSWORD -C -Q 'SELECT 1' || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 40s

networks:
  ia-lab-network:
    external: true
EOF
```

Para ver el contenido del archivo `docker-compose.yml`, ejecuto:

```bash
cat ~/ia-lab/services/motores-bd/mssql/docker-compose.yml
```

Para entrar a editar el archivo, ejecuto:

```bash
sudo nano ~/ia-lab/services/motores-bd/mssql/docker-compose.yml
```

Habilito UFW y permito el puerto `1433` de SQL Server en el host:

```bash
sudo ufw allow 1433/tcp
sudo ufw enable
sudo ufw status
```

**Evidencia (imagen):**


**Registro:**

Creé el archivo `docker-compose.yml` para MS SQL Server utilizando la imagen `mcr.microsoft.com/mssql/server:2022-latest`, el contenedor `mssql-server`, el puerto `1433`, los volúmenes para datos y backups, la red externa `ia-lab-network` y el healthcheck con `sqlcmd`. Verifiqué el contenido del archivo y utilicé espacios reales en la indentación YAML para evitar errores de formato. También habilité el puerto `1433/tcp` en UFW para permitir las conexiones a SQL Server.

### 3.2 Crear el archivo `.env`

Creo el archivo `.env` para configurar MS SQL Server con los datos de mi proyecto `parqueo_agil_control_de_estacionamiento`:

```bash
cat > ~/ia-lab/services/motores-bd/mssql/.env << 'EOF'
TZ=America/Bogota
ACCEPT_EULA=Y
MSSQL_SA_PASSWORD=<CONTRASENA_SEGURA_DE_SA>
MSSQL_PID=Developer
EOF
```

`MSSQL_PID` no es un usuario. Es la edición de SQL Server; en este caso utilizo la edición `Developer`.

El usuario administrador que crea el contenedor se llama `SA` (System Administrator). Por lo tanto, al conectarme mediante `sqlcmd` o DBeaver debo utilizar:

- **Usuario:** `SA`
- **Contraseña:** `<CONTRASENA_SEGURA_DE_SA>`
- **Edición:** `Developer`

Para ver el contenido del archivo `.env`, ejecuto:

```bash
cat ~/ia-lab/services/motores-bd/mssql/.env
```

El contenido esperado es:

```env
TZ=America/Bogota
ACCEPT_EULA=Y
MSSQL_SA_PASSWORD=<CONTRASENA_SEGURA_DE_SA>
MSSQL_PID=Developer
```

Para entrar a editar el archivo `.env`, ejecuto:

```bash
cd ~/ia-lab/services/motores-bd/mssql
sudo nano .env
```

Si necesito cambiar la contraseña, modifico esta línea:

```env
MSSQL_SA_PASSWORD=<CONTRASENA_SEGURA_DE_SA>
```

SQL Server exige que la contraseña de `SA` cumpla los requisitos de complejidad: debe incluir mayúsculas, minúsculas, números y símbolos. Por eso `<TU_CONTRASENA_SEGURA>` sola no es válida y utilizo `<CONTRASENA_SEGURA_DE_SA>`.

Guardo los cambios con `Ctrl + O`, presiono Enter para confirmar y salgo con `Ctrl + X`.

**Evidencia (imagen):**



**Registro:**

Creé y verifiqué el archivo `.env` de MS SQL Server con la zona horaria `America/Bogota`, la aceptación de la licencia mediante `ACCEPT_EULA=Y`, la edición `Developer` y el usuario administrador `SA`. Configuré la contraseña `<CONTRASENA_SEGURA_DE_SA>`, que cumple los requisitos de complejidad exigidos por SQL Server, y comprobé el contenido del archivo con el comando `cat`.

### 3.3 Crear `README.md`

Creo el archivo `README.md` para documentar la configuración de MS SQL Server:

````bash
cat > ~/ia-lab/services/motores-bd/mssql/README.md << 'EOF'
# SQL Server 2022 - Motor de Base de Datos

> **Acceso remoto habilitado.** Puerto expuesto en `0.0.0.0:1433`.
> **Edición (MSSQL_PID):** `Developer`
> **Usuario por defecto:** `SA`
> **Base de datos del proyecto:** `parqueo_agil_control_de_estacionamiento`
> **Password:** `<CONTRASENA_SEGURA_DE_SA>`

---

## Conectar desde WSL (local)

```bash
sudo docker exec -it mssql-server /opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P '<CONTRASENA_SEGURA_DE_SA>' -C
```
EOF
````

Para ver el contenido del archivo `README.md`, ejecuto:

```bash
cat ~/ia-lab/services/motores-bd/mssql/README.md
```

**Evidencia (imagen):**

**Registro:**

Creé el archivo `README.md` de MS SQL Server 2022 y comprobé su contenido con el comando `cat`. Registré el puerto `1433`, la edición `Developer`, el usuario administrador `SA`, la base de datos del proyecto `parqueo_agil_control_de_estacionamiento` y la contraseña `<CONTRASENA_SEGURA_DE_SA>`. También dejé documentado el comando `sqlcmd` para conectarme localmente desde WSL.

### 3.4 Levantar SQL Server

Ingreso a la carpeta de MS SQL Server y levanto el contenedor en segundo plano:

```bash
cd ~/ia-lab/services/motores-bd/mssql
sudo docker compose up -d
```

Verifico que el contenedor `mssql-server` esté corriendo:

```bash
sudo docker ps | grep mssql-server
```

Consulto los últimos 20 mensajes del registro del contenedor:

```bash
sudo docker logs mssql-server --tail 20
```

**Evidencia (imagen):**


**Registro:**

Ingresé a la carpeta de MS SQL Server y levanté el contenedor `mssql-server` en segundo plano con `docker compose up -d`. Después verifiqué que el contenedor estuviera ejecutándose con `docker ps` y consulté los últimos 20 mensajes mediante `docker logs` para comprobar el estado del servicio.

### 3.5 Instalar `mssql-tools` en WSL

Estos pasos instalan `mssql-tools18` y `unixODBC` en Ubuntu 24.04 para administrar y conectarse a SQL Server ejecutado mediante Docker.

#### 3.5.1 Actualizar e instalar los paquetes requeridos

```bash
sudo apt update && sudo apt install -y curl ca-certificates gnupg
```

#### 3.5.2 Eliminar repositorios antiguos o duplicados de Microsoft

Esto evita conflictos con configuraciones anteriores de Ubuntu 22.04 (`jammy`) y con el método obsoleto `apt-key`.

```bash
sudo rm -f /etc/apt/sources.list.d/mssql-release.list
sudo rm -f /etc/apt/sources.list.d/microsoft-prod.list
```

#### 3.5.3 Descargar el repositorio oficial de Microsoft para Ubuntu 24.04

```bash
cd /tmp
curl -sSL -O https://packages.microsoft.com/config/ubuntu/24.04/packages-microsoft-prod.deb
```

#### 3.5.4 Instalar el repositorio oficial de Microsoft

```bash
sudo dpkg -i packages-microsoft-prod.deb
```

#### 3.5.5 Actualizar los repositorios

```bash
sudo apt update
```

#### 3.5.6 Verificar el repositorio de Microsoft

```bash
grep -R "packages.microsoft.com" /etc/apt/sources.list /etc/apt/sources.list.d/ 2>/dev/null
```

#### 3.5.7 Instalar Microsoft SQL Server Tools 18 y unixODBC

```bash
sudo ACCEPT_EULA=Y apt install -y mssql-tools18 unixodbc-dev
```

#### 3.5.8 Agregar `mssql-tools18` al PATH

```bash
echo 'export PATH="$PATH:/opt/mssql-tools18/bin"' >> ~/.bashrc
```

#### 3.5.9 Recargar la configuración de Bash

```bash
source ~/.bashrc
```

#### 3.5.10 Verificar que `sqlcmd` esté instalado

```bash
which sqlcmd
```

El resultado esperado es:

```text
/opt/mssql-tools18/bin/sqlcmd
```
si te parece un error metes esto o lo modificas depende de como tu lo tengas 

```bash 
sudo docker exec -it mssql-server /opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P '<CONTRASENA_SEGURA_DE_SA>' -C
```
**Evidencia (imagen):**




**Registro:**

Seguí los pasos de instalación tal como aparecen en la página web del docente. En WSL instalé los paquetes requeridos, eliminé los repositorios antiguos de Microsoft y configuré el repositorio oficial para Ubuntu 24.04. Después instalé `mssql-tools18` y `unixODBC`, agregué la ruta de las herramientas al `PATH` y recargué la configuración de Bash. Finalmente verifiqué que `sqlcmd` quedó instalado correctamente en `/opt/mssql-tools18/bin/sqlcmd`.

### 3.6 Conectar localmente a SQL Server

Me conecto localmente al contenedor de SQL Server utilizando el usuario administrador `SA` y la contraseña configurada para el proyecto:

```bash
sudo docker exec -it mssql-server /opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P '<CONTRASENA_SEGURA_DE_SA>' -C
```

### 3.6.1 Comandos básicos de referencia

Estos comandos son únicamente de referencia y no los utilizo para este caso durante la instalación ni durante la creación de las tablas de `parqueo_agil_control_de_estacionamiento`.

**Crear una base de datos**

```sql
CREATE DATABASE nombre_bd;
GO
```

**Listar todas las bases de datos**

```sql
SELECT name FROM sys.databases;
GO
```

**Eliminar una base de datos**

No puedo eliminar la base de datos que estoy utilizando. Cambio a `master` y luego ejecuto:

```sql
USE master;
GO
DROP DATABASE nombre_bd;
GO
```

**Entrar a una base de datos**

```sql
USE nombre_bd;
GO
```

**Listar las tablas**

```sql
SELECT name FROM sys.tables;
GO
```

**Listar los campos de una tabla**

```sql
SELECT COLUMN_NAME, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'nombre_tabla';
GO
```

**Salir del motor**

```sql
QUIT
```

### 3.6.2 Crear las tablas del proyecto final

Primero creo la base de datos `parqueo_agil_control_de_estacionamiento`, porque las tablas deben guardarse dentro de una base existente. Este paso se realiza desde `master` y solo es necesario una vez:

```bash
sudo docker exec -it mssql-server /opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P '<CONTRASENA_SEGURA_DE_SA>' -C -d master
```

Cuando aparezca `1>`, escribo:

```sql
CREATE DATABASE parqueo_agil_control_de_estacionamiento;
```

Después presiono Enter, escribo manualmente `GO` y presiono Enter. `GO` no se pega junto con el código SQL; se escribe aparte para ejecutar la instrucción.

La base de datos `parqueo_agil_control_de_estacionamiento` ya fue creada. Para facilitar el proceso y evitar errores al pegar muchas líneas directamente en `sqlcmd`, guardo el código de las 11 tablas en un archivo `.sql` y lo ejecuto desde Bash.

Creo y abro el archivo SQL:

```bash
nano ~/parqueo_agil_control_de_estacionamiento.sql
```

Dentro del archivo pego el código de las 11 tablas que aparece a continuación. Guardo con `Ctrl + O`, presiono Enter y salgo con `Ctrl + X`.

Utilizo espacios reales para la indentación del código, no tabulaciones. Después de salir de `nano`, ejecuto el archivo contra la base de datos `parqueo_agil_control_de_estacionamiento`:

```bash
sudo docker exec -i mssql-server /opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P '<CONTRASENA_SEGURA_DE_SA>' -C -d parqueo_agil_control_de_estacionamiento < ~/parqueo_agil_control_de_estacionamiento.sql
```

El archivo debe contener únicamente las instrucciones SQL de creación de las tablas y cada instrucción debe terminar con `GO`.

Para verificar las tablas, ejecuto desde Bash el comando `sqlcmd` con la base `parqueo_agil_control_de_estacionamiento` y la opción `-Q`.

```sql
CREATE TABLE cliente (
  id INT IDENTITY(1,1) PRIMARY KEY,
  tipo_documento VARCHAR(30) NOT NULL,
  numero_documento VARCHAR(50) NOT NULL UNIQUE,
  nombre VARCHAR(100) NOT NULL,
  telefono VARCHAR(30),
  email VARCHAR(150),
  is_active BIT NOT NULL DEFAULT 1
);
GO

CREATE TABLE estacion (
  id INT IDENTITY(1,1) PRIMARY KEY,
  nombre VARCHAR(100) NOT NULL,
  descripcion VARCHAR(255),
  is_active BIT NOT NULL DEFAULT 1,
  created_at DATETIME2 DEFAULT SYSDATETIME(),
  updated_at DATETIME2 DEFAULT SYSDATETIME()
);
GO

CREATE TABLE bicicleta (
  id INT IDENTITY(1,1) PRIMARY KEY,
  nombre VARCHAR(100) NOT NULL,
  descripcion VARCHAR(255),
  is_active BIT NOT NULL DEFAULT 1,
  created_at DATETIME2 DEFAULT SYSDATETIME(),
  updated_at DATETIME2 DEFAULT SYSDATETIME()
);
GO

CREATE TABLE anclaje (
  id INT IDENTITY(1,1) PRIMARY KEY,
  estacion_id INT NOT NULL,
  bicicleta_id INT,
  nombre VARCHAR(100) NOT NULL,
  descripcion VARCHAR(255),
  is_active BIT NOT NULL DEFAULT 1,
  FOREIGN KEY (estacion_id) REFERENCES estacion(id),
  FOREIGN KEY (bicicleta_id) REFERENCES bicicleta(id)
);
GO

CREATE TABLE tarifa (
  id INT IDENTITY(1,1) PRIMARY KEY,
  nombre VARCHAR(100) NOT NULL,
  regla_calculo VARCHAR(255),
  valor_base DECIMAL(10, 2) NOT NULL,
  vigencia_desde DATE,
  vigencia_hasta DATE,
  is_active BIT NOT NULL DEFAULT 1
);
GO

CREATE TABLE reserva (
  id INT IDENTITY(1,1) PRIMARY KEY,
  cliente_id INT NOT NULL,
  fecha_inicio DATETIME2 NOT NULL,
  fecha_fin DATETIME2,
  estado VARCHAR(30) NOT NULL,
  observaciones VARCHAR(MAX),
  FOREIGN KEY (cliente_id) REFERENCES cliente(id)
);
GO

CREATE TABLE alquiler (
  id INT IDENTITY(1,1) PRIMARY KEY,
  reserva_id INT,
  bicicleta_id INT NOT NULL,
  cliente_id INT NOT NULL,
  fecha_inicio DATETIME2 NOT NULL,
  fecha_fin DATETIME2,
  total DECIMAL(10, 2),
  estado VARCHAR(30) NOT NULL,
  observaciones VARCHAR(MAX),
  FOREIGN KEY (reserva_id) REFERENCES reserva(id),
  FOREIGN KEY (bicicleta_id) REFERENCES bicicleta(id),
  FOREIGN KEY (cliente_id) REFERENCES cliente(id)
);
GO

CREATE TABLE evento_alquiler (
  id INT IDENTITY(1,1) PRIMARY KEY,
  alquiler_id INT NOT NULL,
  tipo VARCHAR(50) NOT NULL,
  fecha DATETIME2 NOT NULL,
  cantidad INT,
  observaciones VARCHAR(MAX),
  estado VARCHAR(30) NOT NULL,
  FOREIGN KEY (alquiler_id) REFERENCES alquiler(id)
);
GO

CREATE TABLE penalidad (
  id INT IDENTITY(1,1) PRIMARY KEY,
  alquiler_id INT NOT NULL,
  tipo VARCHAR(100) NOT NULL,
  fecha DATE NOT NULL,
  valor DECIMAL(10, 2) NOT NULL,
  estado VARCHAR(30) NOT NULL,
  observaciones VARCHAR(MAX),
  FOREIGN KEY (alquiler_id) REFERENCES alquiler(id)
);
GO

CREATE TABLE mantenimiento (
  id INT IDENTITY(1,1) PRIMARY KEY,
  bicicleta_id INT NOT NULL,
  tipo VARCHAR(100) NOT NULL,
  fecha_programada DATE,
  fecha_cierre DATE,
  costo DECIMAL(10, 2),
  estado VARCHAR(30) NOT NULL,
  FOREIGN KEY (bicicleta_id) REFERENCES bicicleta(id)
);
GO

CREATE TABLE pago (
  id INT IDENTITY(1,1) PRIMARY KEY,
  referencia_tipo VARCHAR(50) NOT NULL,
  referencia_id INT NOT NULL,
  metodo VARCHAR(50) NOT NULL,
  monto DECIMAL(10, 2) NOT NULL,
  fecha DATETIME2 NOT NULL,
  estado VARCHAR(30) NOT NULL
);
GO
```

Verifico que las tablas del proyecto se hayan creado correctamente:

```bash
sudo docker exec -it mssql-server /opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P '<CONTRASENA_SEGURA_DE_SA>' -C -d parqueo_agil_control_de_estacionamiento -Q "SELECT name FROM sys.tables;"
```

**Evidencia (imagen):**


**Registro:**

Creé la base de datos `parqueo_agil_control_de_estacionamiento` y ejecuté el archivo `parqueo_agil_control_de_estacionamiento.sql` con las 11 tablas del proyecto final. Utilicé espacios reales en el código para evitar errores al copiar y pegar, ejecuté el archivo directamente desde Bash y verifiqué en SQL Server que las tablas se crearon correctamente.

### 3.7 Crear un usuario propio con acceso remoto

Me conecto primero como `SA` al motor de SQL Server:

```bash
sudo docker exec -it mssql-server /opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P '<CONTRASENA_SEGURA_DE_SA>' -C -d master
```

**Código original de la guía del docente:**

Reemplaza `usuario`, `password` y `nombre_bd` por los valores correspondientes:

```sql
CREATE DATABASE nombre_bd;
GO

CREATE LOGIN usuario WITH PASSWORD = 'password';
GO

USE nombre_bd;
GO
CREATE USER usuario FOR LOGIN usuario;
GO

ALTER ROLE db_owner ADD MEMBER usuario;
GO
```

**Código que colocaré en mi caso:**

La base de datos `parqueo_agil_control_de_estacionamiento` ya existe y contiene las tablas del proyecto. Creo el login `admin`, le asigno el rol `sysadmin` y lo habilito para permitir la administración remota. Ejecuto cada comando por separado:

```sql
CREATE LOGIN admin WITH PASSWORD = '<TU_CONTRASENA_SEGURA>', CHECK_POLICY = OFF;
```

Después de presionar Enter, escribo manualmente `GO` y presiono Enter.

```sql
ALTER SERVER ROLE sysadmin ADD MEMBER admin;
```

Después de presionar Enter, escribo manualmente `GO` y presiono Enter.

```sql
ALTER LOGIN admin ENABLE;
```

Después de presionar Enter, escribo manualmente `GO` y presiono Enter.

`CHECK_POLICY = OFF` desactiva la política de complejidad únicamente para este login; por eso se puede utilizar la contraseña `<TU_CONTRASENA_SEGURA>`. Esta opción no cambia los requisitos de la contraseña de `SA`, que continúa utilizando `<CONTRASENA_SEGURA_DE_SA>` en el archivo `.env`.

**Evidencia (imagen):**

**Registro:**

Al comenzar nuevamente, SQL Server indicó que el login `admin` ya existía. Después ejecuté por separado `ALTER SERVER ROLE sysadmin ADD MEMBER admin;` y `ALTER LOGIN admin ENABLE;`, escribiendo `GO` manualmente después de cada comando. El usuario `admin` quedó habilitado y con permisos de `sysadmin` para administrar remotamente el proyecto `parqueo_agil_control_de_estacionamiento`.

### 3.8 Conectar remotamente desde cualquier equipo

##### **Hacerlo desde DBeaver**

Para conectarme de forma remota a SQL Server utilizo DBeaver. Necesito los siguientes datos:

Obtengo los datos de conexión de la configuración realizada anteriormente:

- La dirección IP del servidor la consulto con `ip a`, buscando la dirección de la interfaz `eth0`.
- El puerto `1433` lo obtengo del archivo `docker-compose.yml`, donde está publicado como `0.0.0.0:1433:1433`, y de la regla de UFW `1433/tcp`.
- La base de datos `parqueo_agil_control_de_estacionamiento` corresponde al nombre utilizado al crear las tablas del proyecto.
- El usuario `admin` y la contraseña `<TU_CONTRASENA_SEGURA>` corresponden al login creado en la sección 3.7.

Consulto la dirección IP del servidor:

```bash
ip a
```

Consulto la edición y la contraseña de `SA` configuradas en el archivo `.env`:

```bash
grep -E '^(MSSQL_PID|MSSQL_SA_PASSWORD)=' ~/ia-lab/services/motores-bd/mssql/.env
```

- **Motor:** SQL Server.
- **Servidor:** `172.18.50.255`.
- **Puerto:** `1433`.
- **Base de datos:** `parqueo_agil_control_de_estacionamiento`.
- **Usuario:** `admin`.
- **Contraseña:** `<TU_CONTRASENA_SEGURA>`.

Antes de abrir DBeaver, compruebo desde WSL que el contenedor esté activo y que el puerto esté publicado:

```bash
sudo docker ps --filter "name=mssql-server" --format "table {{.Names}}\t{{.Ports}}"
```

También compruebo que SQL Server acepte conexiones:

```bash
sudo docker exec mssql-server /opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P '<CONTRASENA_SEGURA_DE_SA>' -C -Q "SELECT 1;"
```

**Evidencia (imagen):**


**Registro:**

Comprobé la dirección IP del servidor, el puerto `1433`, el estado del contenedor `mssql-server` y la respuesta de SQL Server antes de configurar la conexión remota en DBeaver. Estos datos me permitieron confirmar que el servidor estaba listo para recibir la conexión.

En DBeaver realizo los siguientes pasos:

1. Abro DBeaver.
2. Selecciono **Nueva conexión**.
3. Elijo el controlador **SQL Server**.
4. Escribo `172.18.50.255` en **Host**.
5. Escribo `1433` en **Port**.
6. Escribo `admin` en **Username**.
7. Escribo `<TU_CONTRASENA_SEGURA>` en **Password**.
8. Escribo `parqueo_agil_control_de_estacionamiento` en **Database**.
9. Presiono **Test Connection**.
10. Si la prueba es correcta, presiono **Finish** para guardar la conexión.

**Evidencia (imagen):**





**Registro:**

Configuré y probé en DBeaver la conexión remota a SQL Server utilizando el host `172.18.50.255`, el puerto `1433`, la base de datos `parqueo_agil_control_de_estacionamiento` y el usuario `admin` con la contraseña `<TU_CONTRASENA_SEGURA>`.

### 3.9 Backup de una base de datos

**Código original de la guía del docente:**

```bash
sudo docker exec mssql-server /opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P 'MiNiCo57**' -C -Q "BACKUP DATABASE [tecnogua] TO DISK = N'/backups/backup_tecnogua.bak' WITH INIT"
```

**Código que colocaré en mi caso:**

En mi proyecto utilizo la base de datos `parqueo_agil_control_de_estacionamiento`, la contraseña `<CONTRASENA_SEGURA_DE_SA>` y guardo el backup en la carpeta `/backups`, que está conectada con `/mnt/d/academia/bd/`:

```bash
sudo docker exec mssql-server /opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P '<CONTRASENA_SEGURA_DE_SA>' -C -Q "BACKUP DATABASE [parqueo_agil_control_de_estacionamiento] TO DISK = N'/backups/backup_parqueo_agil_control_de_estacionamiento.bak' WITH INIT"
```

Verifico desde WSL que el archivo se haya creado correctamente:

```bash
ls -lh /mnt/d/academia/bd/backup_parqueo_agil_control_de_estacionamiento.bak
```

**Evidencia (imagen):**



**Registro:**

Realicé el backup de la base de datos `parqueo_agil_control_de_estacionamiento` utilizando el usuario `SA` y lo guardé como `backup_parqueo_agil_control_de_estacionamiento.bak` dentro de `/backups`. Después verifiqué desde WSL que el archivo se creó correctamente en `/mnt/d/academia/bd/`.

## 4. Oracle XE

### 4.1 Crear el archivo `docker-compose.yml`

```bash
cat > ~/ia-lab/services/motores-bd/oracle/docker-compose.yml << 'EOF'
services:
  oracle:
    image: gvenzl/oracle-xe:21-slim
    container_name: oracle-server
    restart: unless-stopped

    env_file:
      - .env

    ports:
      - "0.0.0.0:1521:1521"

    volumes:
      - ../../../data/oracle:/opt/oracle/oradata
      - /mnt/d/academia/bd:/backups

    networks:
      - ia-lab-network

    healthcheck:
      test: ["CMD", "healthcheck.sh"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 120s

networks:
  ia-lab-network:
    external: true
EOF
```

Habilito UFW y permito el puerto `1521` de Oracle en el host:

```bash
sudo ufw allow 1521/tcp
sudo ufw enable
sudo ufw status
```

El listener de Oracle XE en `gvenzl/oracle-xe` ya acepta conexiones remotas, equivalente a `--bind-address=0.0.0.0` en MySQL y `listen_addresses=*` en PostgreSQL. No utilizo `command:` aquí, porque en esta imagen sustituiría el entrypoint y el contenedor no arrancaría.

Los usuarios `SYSTEM` y `admin` no se atan a un host, ya que en Oracle no existe `@'%'` como en MySQL. Con el puerto `1521` publicado y UFW abierto, DBeaver puede entrar desde otro equipo utilizando el **Service Name** `parqueo_agil_control_de_estacionamiento`.

**Evidencia (imagen):**


### 4.2 Crear el archivo `.env`

Creo el archivo `.env` para configurar la zona horaria, la contraseña de Oracle XE y el nombre del PDB de mi proyecto: `parqueo_agil_control_de_estacionamiento` con la contraseña `<TU_CONTRASENA_SEGURA>`.

```bash
cat > ~/ia-lab/services/motores-bd/oracle/.env << 'EOF'
TZ=America/Bogota
ORACLE_PASSWORD=<TU_CONTRASENA_SEGURA>
ORACLE_DATABASE=parqueo_agil_control_de_estacionamiento
EOF
```

Para ver lo que guardé dentro del archivo `.env`, ejecuto:

```bash
cat ~/ia-lab/services/motores-bd/oracle/.env
```

`ORACLE_DATABASE` **no es un usuario**. Es el nombre del **PDB** (pluggable database) que crea el contenedor: `parqueo_agil_control_de_estacionamiento`.

El usuario administrador que crea la imagen se llama **`SYSTEM`**; también existe `SYS`. Al conectarme mediante `sqlplus` o DBeaver utilizo `SYSTEM`, no `parqueo_agil_control_de_estacionamiento`.

| Variable | Qué es | Valor |
| --- | --- | --- |
| `ORACLE_DATABASE` | PDB / servicio de conexión | `parqueo_agil_control_de_estacionamiento` |
| `ORACLE_PASSWORD` | Contraseña de `SYSTEM` y `SYS` | `<TU_CONTRASENA_SEGURA>` |

Para editar la clave del archivo `.env` en clase, ingreso a la carpeta de Oracle y abro el archivo:

```bash
cd ~/ia-lab/services/motores-bd/oracle
sudo nano .env
```

Cambio la contraseña por:

```env
ORACLE_PASSWORD=<TU_CONTRASENA_SEGURA>
```

Guardo los cambios con `Ctrl + O` y salgo del editor con `Ctrl + X`.

El primer arranque de Oracle puede tardar varios minutos. La contraseña `<TU_CONTRASENA_SEGURA>` puede ser rechazada por Oracle XE porque el motor exige una contraseña compleja. Si esto ocurre, debo utilizar una contraseña que incluya mayúsculas, minúsculas, números y símbolos.

**Evidencia (imagen):**


**Registro:**

Creé el archivo `.env` de Oracle XE con la zona horaria `America/Bogota`, la contraseña `<TU_CONTRASENA_SEGURA>` y el PDB `parqueo_agil_control_de_estacionamiento`, que son los datos que estoy utilizando actualmente. También dejé documentado que el usuario administrador para conectarme es `SYSTEM` y que `parqueo_agil_control_de_estacionamiento` corresponde al nombre del PDB o servicio de conexión.

### 4.3 Crear `README.md`

Creo el archivo `README.md` para documentar la configuración de Oracle XE:

````bash
cat > ~/ia-lab/services/motores-bd/oracle/README.md << 'EOF'
# Oracle XE - Motor de Base de Datos

> **Acceso remoto habilitado.** Puerto expuesto en `0.0.0.0:1521`.
> **Usuario por defecto:** `SYSTEM`
> **PDB / Service Name:** `parqueo_agil_control_de_estacionamiento`
> **Password:** `<TU_CONTRASENA_SEGURA>`

---

## Conectar desde WSL (local)

```bash
sudo docker exec -it oracle-server sqlplus 'system/<TU_CONTRASENA_SEGURA>@//localhost:1521/parqueo_agil_control_de_estacionamiento'
```
EOF
````

Para comprobar que el archivo se creó correctamente, ejecuto:

```bash
cat ~/ia-lab/services/motores-bd/oracle/README.md
```

**Evidencia (imagen):**


**Registro:**

Creé el archivo `README.md` de Oracle XE utilizando el comando `cat` y después comprobé su contenido con otro comando `cat`. En el archivo documenté el puerto `1521`, el usuario `SYSTEM`, el PDB `parqueo_agil_control_de_estacionamiento`, la contraseña `<TU_CONTRASENA_SEGURA>` y el comando para conectarme localmente desde WSL. Finalmente ejecuté `date` para registrar la fecha de la actividad.

### 4.4 Levantar Oracle

Ingreso a la carpeta de Oracle y levanto el contenedor en segundo plano:

```bash
cd ~/ia-lab/services/motores-bd/oracle
sudo docker compose up -d
```

Verifico si el contenedor `oracle-server` está corriendo y reviso sus últimos 30 mensajes:

```bash
sudo docker ps | grep oracle-server
sudo docker logs oracle-server --tail 30
```

**Evidencia del error (imagen):**


**Registro:**

Ingresé a la carpeta de Oracle XE y ejecuté `sudo docker compose up -d`. La imagen `gvenzl/oracle-xe:21-slim` se descargó correctamente y el contenedor `oracle-server` se inició. Sin embargo, al verificarlo con `sudo docker ps`, observé que aparecía como `Restarting (2)`. Al consultar los registros con `sudo docker logs oracle-server --tail 30`, encontré el error `Cannot create folder` y varios mensajes que indicaban que Oracle no podía crear ni abrir los archivos dentro de `/opt/oracle/oradata/XE/XEPDB1`. El problema estaba relacionado con la carpeta de datos montada desde `~/ia-lab/data/oracle`.

#### 4.4.1 Revisar y corregir los permisos de los datos

Primero reviso los permisos y el contenido de la carpeta de datos:

```bash
ls -ld ~/ia-lab/data/oracle
sudo ls -la ~/ia-lab/data/oracle
```

```bash
sudo docker compose down
```

Cambio el propietario de la carpeta al usuario y grupo utilizados por Oracle dentro del contenedor:

```bash
sudo chown -R 54321:54321 ~/ia-lab/data/oracle
```

Después otorgo permisos de lectura y escritura:

```bash
sudo chmod -R 775 ~/ia-lab/data/oracle
```

Vuelvo a iniciar Oracle:

```bash
sudo docker compose up -d
```


#### 4.4.2 Verificar el inicio de Oracle

Compruebo nuevamente el estado del contenedor y el puerto publicado:

```bash
sudo docker ps | grep oracle-server
```

El resultado esperado es similar a:

```text
oracle-server   Up 10 seconds (health: starting)   0.0.0.0:1521->1521/tcp
```

También reviso los últimos 30 mensajes para comprobar el listener:

```bash
sudo docker logs oracle-server --tail 30
```

En los registros busco el mensaje:

```text
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
```

Este mensaje confirma que el listener de Oracle está escuchando en el puerto `1521` y que el contenedor ya está terminando su inicialización.

**Evidencia de la solución (imagen):**


**Registro final:**

Después de corregir los permisos, volví a iniciar Oracle XE. El contenedor `oracle-server` dejó de reiniciarse y apareció como `Up (healthy)`. También comprobé que el puerto `1521` estaba publicado y que el listener escuchaba en `0.0.0.0:1521`. En los registros confirmé que el PDB `parqueo_agil_control_de_estacionamiento` se abrió en modo lectura y escritura con el mensaje `Pluggable database parqueo_agil_control_de_estacionamiento opened read write`. Finalmente ejecuté `date` y registré la fecha de la actividad.

### 4.5 Conectar localmente a Oracle

Ingreso al contenedor de Oracle XE:

```bash
sudo docker exec -it oracle-server bash
```

Me conecto a Oracle utilizando el usuario administrador `SYSTEM`, la contraseña `<TU_CONTRASENA_SEGURA>` y el PDB `parqueo_agil_control_de_estacionamiento`:

```bash
sqlplus 'system/<TU_CONTRASENA_SEGURA>@//localhost:1521/parqueo_agil_control_de_estacionamiento'
```

Debo esperar a que aparezca el indicador `SQL>` antes de escribir comandos de Oracle. Entrar al contenedor solo muestra el aviso `bash-4.4$`; en ese punto todavía estoy en Bash y no en SQL*Plus.

Si escribo comandos como `SELECT`, `CONN`, `DESC` o `EXIT` mientras aparece `bash-4.4$`, Bash intenta interpretarlos como comandos de Linux y muestra un error. Para ejecutar esos comandos debo estar dentro de SQL*Plus, donde el aviso es `SQL>`.

Ya dentro del motor, creo el usuario `admin`, que utilizaré como esquema del proyecto `parqueo_agil_control_de_estacionamiento`:

```sql
CREATE USER admin IDENTIFIED BY "<TU_CONTRASENA_SEGURA>" DEFAULT TABLESPACE USERS QUOTA UNLIMITED ON USERS;
ALTER USER admin QUOTA UNLIMITED ON USERS;
GRANT CONNECT, RESOURCE TO admin;
```

**Evidencia (imagen):**



**Registro:**

Ingresé al contenedor `oracle-server` y me conecté localmente al PDB `parqueo_agil_control_de_estacionamiento` con el usuario `SYSTEM`. Después creé el usuario `admin`, le asigné una cuota ilimitada en el tablespace `USERS` y le otorgué los privilegios `CONNECT` y `RESOURCE`. Oracle confirmó las operaciones con los mensajes `User created`, `User altered` y `Grant succeeded`. Finalmente salí de SQL*Plus, salí del contenedor y ejecuté `date` para registrar la fecha de la actividad.

### 4.5.1 Comandos normales de Oracle

En Oracle el PDB `parqueo_agil_control_de_estacionamiento` es la base de datos y cada usuario funciona como un esquema. Por eso trabajo dentro del PDB `parqueo_agil_control_de_estacionamiento` y utilizo el usuario `admin` para los objetos de mi proyecto.

**Listar todos los usuarios o esquemas**

```sql
SELECT username FROM all_users ORDER BY username;
```

**Eliminar el esquema del proyecto**

Si necesito eliminar el esquema `admin`, me conecto como `SYS` o `SYSTEM` y ejecuto:

```sql
DROP USER admin CASCADE;
```

No elimino el usuario `SYSTEM`, porque es el administrador que utilizo para conectarme al PDB `parqueo_agil_control_de_estacionamiento`.

**Entrar al esquema del proyecto**

Me conecto al usuario `admin` dentro del PDB `parqueo_agil_control_de_estacionamiento`:

```sql
CONN admin/<TU_CONTRASENA_SEGURA>@//localhost:1521/parqueo_agil_control_de_estacionamiento
```

**Listar las tablas**

```sql
SELECT owner, table_name
FROM all_tables
WHERE owner = 'ADMIN'
ORDER BY table_name;
```

**Listar los campos de una tabla**

```sql
DESC nombre_tabla;
```

También consulto los campos con:

```sql
SELECT column_name, data_type, data_length
FROM user_tab_columns
WHERE table_name = 'NOMBRE_TABLA';
```

**Salir del motor**

```sql
EXIT;
```

### 4.6 Crear un usuario propio con acceso remoto

Para crear y administrar mi usuario propio, primero me conecto como `SYSTEM` al PDB `parqueo_agil_control_de_estacionamiento`:

```bash
sudo docker exec -it oracle-server bash
sqlplus 'system/<TU_CONTRASENA_SEGURA>@//localhost:1521/parqueo_agil_control_de_estacionamiento'
```

En mi caso, el usuario `admin` ya fue creado en el paso 4.5 con la contraseña `<TU_CONTRASENA_SEGURA>`, cuota ilimitada sobre `USERS` y los privilegios `CONNECT` y `RESOURCE`.

Oracle no utiliza usuarios asociados a un host mediante `@'%'`, como MySQL. Para permitir la conexión remota utilizo el puerto publicado `1521`, el listener activo y el Service Name `parqueo_agil_control_de_estacionamiento`.

Para conectarme desde otro equipo con DBeaver utilizo los siguientes datos:

- **Motor:** Oracle.
- **Host:** la dirección IP del equipo donde está ejecutándose Docker.
- **Port:** `1521`.
- **Service Name:** `parqueo_agil_control_de_estacionamiento`.
- **Username:** `admin`.
- **Password:** `<TU_CONTRASENA_SEGURA>`.

Me conecto al esquema `admin` y consulto la información del proyecto:

```sql
CONN admin/<TU_CONTRASENA_SEGURA>@//localhost:1521/parqueo_agil_control_de_estacionamiento
SELECT username FROM all_users ORDER BY username;
SELECT owner, table_name FROM all_tables WHERE owner = 'ADMIN' ORDER BY table_name;
DESC nombre_tabla;
EXIT;
```

**Evidencia del intento (imagen):**



**Registro:**

En este intento entré al contenedor, pero no llegué a SQL*Plus, porque no apareció el indicador `SQL>`. Por esa razón, Bash intentó interpretar `CONN`, `SELECT`, `DESC` y `EXIT` como comandos de Linux y mostró el mensaje `command not found`. Después salí del contenedor y ejecuté `sqlplus` directamente en Ubuntu, donde también apareció `sqlplus: command not found`.

#### Procedimiento correcto para repetirlo

Para hacerlo nuevamente, ejecuto los comandos paso a paso y espero el indicador correspondiente antes de continuar:

**Paso 1. Entro al contenedor:**

```bash
sudo docker exec -it oracle-server bash
```

Debo comprobar que aparezca `bash-4.4$`.

**Paso 2. Inicio SQL*Plus como `SYSTEM`:**

```bash
sqlplus 'system/<TU_CONTRASENA_SEGURA>@//localhost:1521/parqueo_agil_control_de_estacionamiento'
```

Espero a que aparezca `SQL>`. Los siguientes comandos solo se ejecutan después de ver ese indicador.

**Paso 3. Me conecto al usuario `admin`:**

```sql
CONN admin/<TU_CONTRASENA_SEGURA>@//localhost:1521/parqueo_agil_control_de_estacionamiento
```

**Paso 4. Consulto los usuarios o esquemas:**

```sql
SELECT username FROM all_users ORDER BY username;
```

**Paso 5. Consulto las tablas del usuario `admin`:**

```sql
SELECT owner, table_name
FROM all_tables
WHERE owner = 'ADMIN'
ORDER BY table_name;
```

**Registro del resultado:**

Después de iniciar SQL*Plus como `SYSTEM`, me conecté correctamente al usuario `admin` dentro del PDB `parqueo_agil_control_de_estacionamiento`. La consulta de usuarios mostró 28 usuarios, incluido `ADMIN`. Al consultar las tablas del usuario `admin`, Oracle respondió `no rows selected`, lo que confirma que el esquema está creado correctamente, pero todavía no contiene tablas.

### 4.5.2 Crear las tablas del proyecto parqueo_agil_control_de_estacionamiento

La imagen corresponde a mi proyecto final **parqueo_agil_control_de_estacionamiento - Bicicletas compartidas**. Como el esquema `admin` todavía no tenía tablas, creo las 11 tablas del modelo dentro del PDB `parqueo_agil_control_de_estacionamiento`.

**Evidencia del modelo del proyecto (imagen):**


Antes de copiar el código de las tablas, entro al contenedor desde Ubuntu:

```bash
sudo docker exec -it oracle-server bash
```

Dentro del contenedor inicio SQL*Plus como `SYSTEM`:

```bash
sqlplus 'system/<TU_CONTRASENA_SEGURA>@//localhost:1521/parqueo_agil_control_de_estacionamiento'
```

Cuando aparezca el indicador `SQL>`, me conecto al esquema `admin`:

```sql
CONN admin/<TU_CONTRASENA_SEGURA>@//localhost:1521/parqueo_agil_control_de_estacionamiento
```

Después de que aparezca nuevamente `SQL>`, ejecuto las siguientes instrucciones para crear las tablas:

```sql
CREATE TABLE cliente (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  tipo_documento VARCHAR2(30) NOT NULL,
  numero_documento VARCHAR2(50) NOT NULL UNIQUE,
  nombre VARCHAR2(100) NOT NULL,
  telefono VARCHAR2(30),
  email VARCHAR2(150),
  is_active NUMBER(1) DEFAULT 1 NOT NULL
);

CREATE TABLE estacion (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  nombre VARCHAR2(100) NOT NULL,
  descripcion VARCHAR2(255),
  is_active NUMBER(1) DEFAULT 1 NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE bicicleta (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  nombre VARCHAR2(100) NOT NULL,
  descripcion VARCHAR2(255),
  is_active NUMBER(1) DEFAULT 1 NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE anclaje (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  estacion_id NUMBER NOT NULL,
  bicicleta_id NUMBER,
  nombre VARCHAR2(100) NOT NULL,
  descripcion VARCHAR2(255),
  is_active NUMBER(1) DEFAULT 1 NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_anclaje_estacion FOREIGN KEY (estacion_id) REFERENCES estacion(id),
  CONSTRAINT fk_anclaje_bicicleta FOREIGN KEY (bicicleta_id) REFERENCES bicicleta(id)
);

CREATE TABLE tarifa (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  nombre VARCHAR2(100) NOT NULL,
  regla_calculo VARCHAR2(255),
  valor_base NUMBER(10, 2) NOT NULL,
  vigencia_desde DATE,
  vigencia_hasta DATE,
  is_active NUMBER(1) DEFAULT 1 NOT NULL
);

CREATE TABLE reserva (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  cliente_id NUMBER NOT NULL,
  fecha_inicio TIMESTAMP NOT NULL,
  fecha_fin TIMESTAMP,
  estado VARCHAR2(30) NOT NULL,
  observaciones VARCHAR2(1000),
  CONSTRAINT fk_reserva_cliente FOREIGN KEY (cliente_id) REFERENCES cliente(id)
);

CREATE TABLE alquiler (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  reserva_id NUMBER,
  bicicleta_id NUMBER NOT NULL,
  cliente_id NUMBER NOT NULL,
  fecha_inicio TIMESTAMP NOT NULL,
  fecha_fin TIMESTAMP,
  total NUMBER(10, 2),
  estado VARCHAR2(30) NOT NULL,
  observaciones VARCHAR2(1000),
  CONSTRAINT fk_alquiler_reserva FOREIGN KEY (reserva_id) REFERENCES reserva(id),
  CONSTRAINT fk_alquiler_bicicleta FOREIGN KEY (bicicleta_id) REFERENCES bicicleta(id),
  CONSTRAINT fk_alquiler_cliente FOREIGN KEY (cliente_id) REFERENCES cliente(id)
);

CREATE TABLE evento_alquiler (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  alquiler_id NUMBER NOT NULL,
  tipo VARCHAR2(50) NOT NULL,
  fecha TIMESTAMP NOT NULL,
  cantidad NUMBER,
  observaciones VARCHAR2(1000),
  estado VARCHAR2(30) NOT NULL,
  CONSTRAINT fk_evento_alquiler FOREIGN KEY (alquiler_id) REFERENCES alquiler(id)
);

CREATE TABLE penalidad (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  alquiler_id NUMBER NOT NULL,
  tipo VARCHAR2(100) NOT NULL,
  fecha DATE NOT NULL,
  valor NUMBER(10, 2) NOT NULL,
  estado VARCHAR2(30) NOT NULL,
  observaciones VARCHAR2(1000),
  CONSTRAINT fk_penalidad_alquiler FOREIGN KEY (alquiler_id) REFERENCES alquiler(id)
);

CREATE TABLE mantenimiento (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  bicicleta_id NUMBER NOT NULL,
  tipo VARCHAR2(100) NOT NULL,
  fecha_programada DATE,
  fecha_cierre DATE,
  costo NUMBER(10, 2),
  estado VARCHAR2(30) NOT NULL,
  CONSTRAINT fk_mantenimiento_bicicleta FOREIGN KEY (bicicleta_id) REFERENCES bicicleta(id)
);

CREATE TABLE pago (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  referencia_tipo VARCHAR2(50) NOT NULL,
  referencia_id NUMBER NOT NULL,
  metodo VARCHAR2(50) NOT NULL,
  monto NUMBER(10, 2) NOT NULL,
  fecha TIMESTAMP NOT NULL,
  estado VARCHAR2(30) NOT NULL
);
```

Después de ejecutar cada instrucción, Oracle debe mostrar el mensaje `Table created.`. Compruebo que las tablas del esquema `admin` quedaron creadas:

```sql
SELECT owner, table_name
FROM all_tables
WHERE owner = 'ADMIN'
ORDER BY table_name;
```

**Evidencia (imágenes):**



**Registro:**

Como la consulta anterior mostró `no rows selected`, utilicé la imagen de mi proyecto final **parqueo_agil_control_de_estacionamiento - Bicicletas compartidas** para definir las tablas que necesito en Oracle. Ejecuté el código con las 11 entidades del modelo y sus relaciones principales dentro del esquema `admin`. Después consulté `all_tables` filtrando el propietario `ADMIN` y Oracle mostró 11 filas, por lo que confirmé que las tablas quedaron creadas correctamente. Finalmente salí de SQL*Plus, salí del contenedor y ejecuté `date` para registrar la fecha de la actividad.

### 4.7 Conectar remotamente desde cualquier equipo

##### **Hacerlo desde DBeaver**

Para conectarme remotamente a Oracle XE utilizo DBeaver. Primero consulto la dirección IP del equipo donde está ejecutándose Docker:

```bash
ip a
```

En el resultado busco la interfaz `eth0`. En mi caso, utilizo la dirección IP `172.18.50.255` como **Host**.

También verifico que el contenedor esté activo y que el puerto `1521` esté publicado:

```bash
sudo docker ps --filter "name=oracle-server" --format "table {{.Names}}\t{{.Ports}}"
```

Compruebo que el puerto `1521` esté permitido por UFW:

```bash
sudo ufw status
```

**Evidencia (imagen):**


**Registro:**

Consulté la configuración de red con `ip a` y comprobé que la interfaz `eth0` tiene la dirección IP `172.18.50.255`. También verifiqué que el contenedor `oracle-server` está activo y publica el puerto `1521` mediante `0.0.0.0:1521->1521/tcp`. Finalmente comprobé con `sudo ufw status` que el firewall está activo y que el puerto `1521/tcp` está permitido para IPv4 e IPv6. Ejecuté `date` para registrar la fecha de la actividad.

En DBeaver realizo los siguientes pasos:

1. Abro DBeaver.
2. Selecciono **Nueva conexión**.
3. Elijo el controlador **Oracle**.

En la pestaña **Basic** dejo la conexión con estos valores:

4. Escribo `172.18.50.255` en **Host**.
5. Escribo `1521` en **Port**.
6. En el campo **Database**, escribo `parqueo_agil_control_de_estacionamiento`.
7. En el selector que aparece a la derecha de **Database**, selecciono **Service Name**.
8. Escribo `admin` en **Nombre de usuario**.
9. Escribo `<TU_CONTRASENA_SEGURA>` en **Contraseña**.
10. En **Role**, selecciono **Normal**.
11. Dejo **Authentication** en **Username/password**.
12. Dejo **Local Client** en `<not present>`.
13. Presiono **Probar conexión...**.
14. Si la prueba es correcta, presiono **Finalizar** para guardar la conexión.

En **Role** aparecen tres opciones:

- **Normal:** es la opción que utilizo con el usuario `admin`.
- **SYSDBA:** la utilizo únicamente si me conecto con `SYS` para tareas administrativas.
- **SYSOPER:** la utilizo únicamente para operaciones especiales de administración de la instancia.

Para mi conexión no selecciono **SYSDBA** ni **SYSOPER**.

Para la conexión remota utilizo los siguientes valores:

- **Motor:** Oracle.
- **Host:** `172.18.50.255`.
- **Port:** `1521`.
- **Service Name:** `parqueo_agil_control_de_estacionamiento`.
- **Username:** `admin`.
- **Password:** `<TU_CONTRASENA_SEGURA>`.

En Oracle utilizo **Service Name** en lugar de **Database**, porque `parqueo_agil_control_de_estacionamiento` es el nombre del PDB y del servicio de conexión.

**Evidencia (imágenes):**



**Registro:**

Consulté la dirección IP del equipo mediante `ip a` y utilicé `172.18.50.255` como Host. Después comprobé que el contenedor `oracle-server` estuviera activo, que el puerto `1521` estuviera publicado y que UFW permitiera ese puerto. Finalmente configuré en DBeaver una conexión Oracle con el Service Name `parqueo_agil_control_de_estacionamiento`, el usuario `admin` y la contraseña `<TU_CONTRASENA_SEGURA>`. La prueba fue exitosa y DBeaver mostró el mensaje `Conectado`; también verifiqué los detalles de la conexión y observé la conexión `parqueo_agil_control_de_estacionamiento Oracle` en el panel de conexiones.

### 4.8 Backup de una base de datos

Realizo un backup del PDB `parqueo_agil_control_de_estacionamiento` utilizando Oracle Data Pump y el usuario administrador `SYSTEM`:

```bash
sudo docker exec oracle-server expdp system/<TU_CONTRASENA_SEGURA>@parqueo_agil_control_de_estacionamiento directory=DATA_PUMP_DIR dumpfile=backup_parqueo_agil_control_de_estacionamiento.dmp logfile=backup_parqueo_agil_control_de_estacionamiento.log
```

El comando genera el archivo `backup_parqueo_agil_control_de_estacionamiento.dmp` y el registro `backup_parqueo_agil_control_de_estacionamiento.log` dentro del directorio de Data Pump configurado por Oracle.

**Evidencia del primer intento (imagen):**


**Registro:**

Realicé el primer intento de backup del PDB `parqueo_agil_control_de_estacionamiento` utilizando el usuario `SYSTEM`, la contraseña `<TU_CONTRASENA_SEGURA>` y el directorio `DATA_PUMP_DIR`. El comando no terminó correctamente: Oracle mostró `ORA-12154: TNS:could not resolve the connect identifier specified`, porque `@parqueo_agil_control_de_estacionamiento` se interpretó como un alias TNS que no estaba configurado. Por esta razón, en este intento no se confirmó la creación del archivo `backup_parqueo_agil_control_de_estacionamiento.dmp`.

Para repetir el backup utilizando la conexión EZCONNECT, ejecuto:

```bash
sudo docker exec oracle-server expdp 'system/<TU_CONTRASENA_SEGURA>@//localhost:1521/parqueo_agil_control_de_estacionamiento' directory=DATA_PUMP_DIR dumpfile=backup_parqueo_agil_control_de_estacionamiento.dmp logfile=backup_parqueo_agil_control_de_estacionamiento.log
```

**Evidencia del segundo intento (imagen):**


**Registro del segundo intento:**

Después de corregir la cadena de conexión y utilizar EZCONNECT, ejecuté nuevamente el backup de `parqueo_agil_control_de_estacionamiento`. Oracle confirmó la conexión con la base de datos, procesó los objetos del esquema y generó el archivo `backup_parqueo_agil_control_de_estacionamiento.dmp` dentro de `DATA_PUMP_DIR`. El proceso terminó correctamente con el mensaje `Job ... completed` y finalmente ejecuté `date` para registrar la fecha de la actividad.

### 4.9 Variables clave del `.env`

Estas son las variables principales que configuré para Oracle XE:

| Variable | Descripción | Valor |
| --- | --- | --- |
| `TZ` | Zona horaria | `America/Bogota` |
| `ORACLE_PASSWORD` | Contraseña de `SYSTEM` y `SYS` | `<TU_CONTRASENA_SEGURA>` |
| `ORACLE_DATABASE` | PDB inicial / Service Name | `parqueo_agil_control_de_estacionamiento` |

Para consultar nuevamente estas variables, ejecuto:

```bash
cat ~/ia-lab/services/motores-bd/oracle/.env
```

**Evidencia (imagen):**


**Registro:**

Consulté el contenido del archivo `.env` con el comando `cat` y confirmé que contiene `TZ=America/Bogota`, `ORACLE_PASSWORD=<TU_CONTRASENA_SEGURA>` y `ORACLE_DATABASE=parqueo_agil_control_de_estacionamiento`. También ejecuté `date` y registré la fecha de la actividad.

# Conclusión

Durante el desarrollo de este taller instalé, configuré y verifiqué satisfactoriamente los cuatro motores de bases de datos solicitados: MySQL, PostgreSQL, MS SQL Server y Oracle XE. En cada motor creé la base de datos `parqueo_agil_control_de_estacionamiento`, configuré los contenedores Docker, revisé los registros de ejecución, habilité los puertos correspondientes y comprobé las conexiones locales y remotas mediante DBeaver.

El proceso me permitió resolver diferentes situaciones técnicas, como errores de indentación en archivos YAML, permisos de las carpetas de datos de Oracle, configuración de usuarios y diferencias entre los comandos de cada motor. También creé y verifiqué las tablas del proyecto **parqueo_agil_control_de_estacionamiento - Bicicletas compartidas**, manteniendo sus relaciones principales en los cuatro motores. Finalmente confirmé que los servicios quedaron activos y disponibles para trabajar con la información del proyecto.

Este informe de trazabilidad reúne los comandos utilizados, las configuraciones aplicadas, los problemas encontrados, las soluciones implementadas y las evidencias obtenidas durante el proceso.


Estudiante de Ingeniería de Sistemas
alvaro javier montero fonseca  
Facultad de Ingeniería  
Universidad de La Guajira     
Fecha del informe de trazabilidad: 9 de septiembre de 2026



