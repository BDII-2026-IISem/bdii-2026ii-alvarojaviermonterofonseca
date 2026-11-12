# Parqueo Ágil · Guías de Bases de Datos

Documentación técnica para instalar, configurar y administrar el proyecto **Parqueo Ágil** con cuatro motores de bases de datos:

- MySQL 8.0
- PostgreSQL 17
- Microsoft SQL Server 2022
- Oracle Database XE 21c

## Documentación

| Guía | Descripción |
| --- | --- |
| [Instalación inicial](docs/01-instalacion-inicial/README.md) | Configuración documentada de Docker, los motores de base de datos, conexiones y respaldos. |
| [Segunda instalación](docs/02-instalacion-secundaria/segunda-instalacion.md) | Instancia independiente con contenedores, puertos, red y datos propios. |
| [Guía unificada](docs/03-guia-unificada/guia-unificada.md) | Referencia consolidada para las instalaciones original y secundaria. |
| [Creación manual de tablas](docs/04-creacion-manual-tablas/mysql.md) | Guía visual para crear tablas desde MySQL Workbench. |

## Estructura

```text
docs/
├── 01-instalacion-inicial/       # Guía principal y sus evidencias
├── 02-instalacion-secundaria/    # Segunda instancia aislada
├── 03-guia-unificada/            # Referencia completa
└── 04-creacion-manual-tablas/    # Procedimiento en MySQL Workbench
```

## Antes de empezar

Las instrucciones se ejecutan en **Ubuntu/WSL** con Docker y Docker Compose disponibles. Las contraseñas y los archivos `.env` no deben subirse al repositorio.

## Seguridad

Las claves de ejemplo se sustituyeron por marcadores. Define tus propias contraseñas en archivos .env locales y nunca las publiques. Las capturas de la instalación inicial se excluyeron para evitar exponer datos de configuración.

---

> Las capturas de creación manual de tablas se incluyen como evidencia del procedimiento realizado.
