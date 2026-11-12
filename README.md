# Parqueo Ágil · Bases de Datos

Documentación académica del proyecto **Parqueo Ágil**, una aplicación de control de estacionamiento. El repositorio reúne las guías de instalación y administración de cuatro motores de bases de datos, el informe técnico del backend en Django y sus evidencias.

| Dato | Información |
| --- | --- |
| Universidad | Universidad de La Guajira |
| Programa | Ingeniería de Sistemas |
| Área | Base de Datos II |
| Periodo | 2026-II |
| Estudiante | Alvaro Montero |
| Docente | Ing. Jaider J. Quintero Mendoza, PhD |
| Proyecto | Parqueo Ágil: control de estacionamiento |

## Contenido del repositorio

| Carpeta | Contenido |
| --- | --- |
| [`informe base de datos alvaro/`](informe%20base%20de%20datos%20alvaro/) | Informe principal: instalación de Docker y los cuatro motores, segunda instalación, guía unificada y consultas avanzadas. |
| [`parqueo-agil-base-de-datos/`](parqueo-agil-base-de-datos/README.md) | Documentación organizada por etapas: instalación inicial, segunda instalación, guía unificada y creación manual de tablas. |
| [`informe de django/`](informe%20de%20django/evidencia/Informe-Django.md) | Informe técnico del proyecto Django (ISS-00 a ISS-13) con sus evidencias. |
| [`creaciondetablasmanual/`](creaciondetablasmanual/mysql/msql.md) | Guía visual para crear tablas desde MySQL Workbench. |
| [`base de datos 2/`](base%20de%20datos%202/%23%20Instalaci%C3%B3n%20y%20paso%20a%20paso%20de%20modelos%20d.md) | Documento de referencia que enlaza a la guía principal de instalación. |

> La carpeta `parqueo-agil-base-de-datos/docs/` contiene una versión organizada por etapas de las mismas guías. El informe principal se conserva en `informe base de datos alvaro/`.

## Guías de bases de datos

| Guía | Descripción |
| --- | --- |
| [Instalación y paso a paso de modelos de datos](informe%20base%20de%20datos%20alvaro/%23%20Instalaci%C3%B3n%20y%20paso%20a%20paso%20de%20modelos%20d.md) | Docker, MySQL 8.0, PostgreSQL 17, SQL Server 2022 y Oracle XE 21c: contenedores, usuarios, conexiones locales y remotas, respaldos y variables de entorno. |
| [Segunda instalación independiente](informe%20base%20de%20datos%20alvaro/guiasegundobasededatos.md) | Creación de `parqueo_agil_control_de_estacionamientoII` con contenedores, puertos, red y datos propios. |
| [Guía unificada](informe%20base%20de%20datos%20alvaro/guajia.md) | Referencia consolidada de las dos instalaciones, migración de datos entre motores y diagnóstico de contenedores. |
| [Consultas avanzadas en MySQL](informe%20base%20de%20datos%20alvaro/busquedas_avanzadas/consultas_avanzadas_parqueoagil.md) | 18 consultas para Parqueo Ágil: JOIN, subconsultas, rankings, transacciones, procedimientos almacenados y optimización con índices. |
| [Creación manual de tablas](creaciondetablasmanual/mysql/msql.md) | Procedimiento visual para crear un esquema y tablas desde MySQL Workbench. |

## Informe de Django

El [informe técnico del proyecto Django](informe%20de%20django/evidencia/Informe-Django.md) documenta la construcción del backend de Parqueo Ágil:

- **ISS-00:** entorno, proyecto Django y `.gitignore`.
- **ISS-01:** settings, `.env` y selección del motor de base de datos.
- **ISS-02 a ISS-06:** apps de clientes, seguridad, catálogo y operación de tickets.
- **ISS-07 a ISS-12:** REST Framework, CORS, serializers en camelCase, ViewSets, disponibilidad de espacios, ingreso y cierre de tickets.
- **ISS-13:** configuración del Django Admin.

## Estructura

```text
.
├── README.md
├── .gitignore
├── informe base de datos alvaro/
│   ├── # Instalación y paso a paso de modelos d.md
│   ├── guajia.md
│   ├── guiasegundobasededatos.md
│   └── busquedas_avanzadas/
├── parqueo-agil-base-de-datos/
│   ├── README.md
│   └── docs/
│       ├── 01-instalacion-inicial/
│       ├── 02-instalacion-secundaria/
│       ├── 03-guia-unificada/
│       └── 04-creacion-manual-tablas/
├── informe de django/
│   └── evidencia/
├── creaciondetablasmanual/
│   └── mysql/
└── base de datos 2/
```

## Requisitos

- Ubuntu/WSL2 con Docker Engine y Docker Compose.
- DBeaver y/o MySQL Workbench para las conexiones y la administración visual.
- Python 3.12+ y Django 5.2.17 para reproducir el informe de Django.
- Conocimientos básicos de SQL y de contenedores.

## Seguridad

- No se incluyen archivos `.env`, contraseñas ni credenciales reales.
- Los ejemplos usan marcadores como `<TU_CONTRASENA_SEGURA>`; define tus propias contraseñas en un `.env` local.
- Las capturas que podían exponer configuración local o contraseñas se mantienen fuera del repositorio.
- El `.gitignore` excluye entornos virtuales, cachés, logs, respaldos y archivos de configuración local.

> Las guías se ejecutan en Ubuntu/WSL. No ejecutes los comandos de Docker desde PowerShell ni desde el editor SQL de DBeaver.
