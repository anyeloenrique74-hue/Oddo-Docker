# Despliegue de Odoo 19 con Docker Compose

Despliegue local de **Odoo 19.0 (Community)** con **PostgreSQL 16**, usando Docker Compose. Odoo se sirve en **http://localhost:8010**.

## Estructura del proyecto

```
Oddo-Docker/
├── docker-compose.yml   # Definición de los servicios db y odoo
├── config/
│   └── odoo.conf        # Configuración de Odoo (montada en /etc/odoo)
└── addons/              # Módulos propios (montada en /mnt/extra-addons)
```

## Requisitos

- Docker Desktop (con Docker Compose v2).
- El puerto **8010** libre en el equipo anfitrión.

## Arquitectura

```
 Navegador ──► localhost:8010 ──► [ odoo  (odoo:19.0) :8069 ] ──► [ db (postgres:16) :5432 ]
                                        │                               │
                              volumen odoo-web-data            volumen odoo-db-data
                              ./config  → /etc/odoo
                              ./addons  → /mnt/extra-addons
```

Los dos contenedores comparten la red `oddo-docker_default` que crea Compose. Odoo encuentra la base de datos por el nombre del servicio (`db`). PostgreSQL **no** publica su puerto al anfitrión: solo Odoo puede acceder a él.

## Configuración

### Servicio `db` (PostgreSQL 16)

| Parámetro | Valor | Propósito |
|---|---|---|
| Imagen | `postgres:16` | Motor de base de datos |
| `POSTGRES_USER` / `POSTGRES_PASSWORD` | `odoo` / `odoo` | Usuario con el que se conecta Odoo |
| `POSTGRES_DB` | `postgres` | Base inicial; Odoo crea sus propias bases |
| `PGDATA` | `/var/lib/postgresql/data/pgdata` | Subcarpeta de datos dentro del volumen |
| Volumen | `odoo-db-data` | Persistencia de las bases de datos |
| `healthcheck` | `pg_isready` cada 10 s, 5 reintentos | Indica cuándo PostgreSQL acepta conexiones |
| `restart` | `unless-stopped` | Se reinicia solo, salvo que se detenga a mano |

### Servicio `odoo` (Odoo 19.0)

| Parámetro | Valor | Propósito |
|---|---|---|
| Imagen | `odoo:19.0` | Imagen oficial de Odoo Community |
| Puertos | `8010:8069` | Odoo escucha en 8069 dentro del contenedor y se publica en el 8010 del anfitrión |
| `HOST` / `USER` / `PASSWORD` | `db` / `odoo` / `odoo` | Conexión a PostgreSQL (el entrypoint de la imagen los convierte en `--db_host`, `--db_user`, `--db_password`) |
| `depends_on` | `db` | Arranca después del contenedor de base de datos |
| Volumen `odoo-web-data` | `/var/lib/odoo` | Filestore (adjuntos) y sesiones |
| Bind mount `./config` | `/etc/odoo` | Reemplaza la configuración por defecto de la imagen |
| Bind mount `./addons` | `/mnt/extra-addons` | Módulos propios |
| `restart` | `unless-stopped` | Igual que `db` |

### `config/odoo.conf`

```ini
[options]
addons_path = /mnt/extra-addons
admin_passwd = <contraseña maestra>
```

- **`addons_path`**: agrega la carpeta `addons/` del proyecto a las rutas de módulos. Los módulos estándar de Odoo se siguen cargando.
- **`admin_passwd`**: es la **contraseña maestra**. El gestor de bases de datos (`/web/database/manager`) la pide para crear, duplicar, respaldar, restaurar o eliminar bases. No es la contraseña del usuario administrador de Odoo.

Los datos de conexión a PostgreSQL no están en este archivo porque llegan por las variables de entorno del servicio `odoo`.

> **Nota:** este `odoo.conf` reemplaza al de la imagen y no define `data_dir`. Por eso Odoo guarda el filestore en `/var/lib/odoo/.local/share/Odoo/filestore/`, que sigue dentro del volumen `odoo-web-data`, así que los datos persisten.

### Volúmenes

| Volumen (nombre real en Docker) | Contenido |
|---|---|
| `oddo-docker_odoo-db-data` | Datos de PostgreSQL (todas las bases de Odoo) |
| `oddo-docker_odoo-web-data` | Filestore y sesiones de Odoo |

El prefijo `oddo-docker_` viene del nombre de la carpeta del proyecto.

## Puesta en marcha

Desde la carpeta del proyecto:

```bash
docker compose up -d
```

La primera vez se descargan las imágenes. Después:

1. Abre **http://localhost:8010**.
2. Si no existe ninguna base de datos, Odoo muestra el formulario para crearla. Escribe la contraseña maestra (`admin_passwd`), el nombre de la base, el correo y la contraseña del administrador, el idioma y el país.
3. Inicia sesión con el correo y la contraseña del administrador que acabas de definir.

Estado actual: ya existe la base **`odoo_docker`**.

## Comandos útiles

| Acción | Comando |
|---|---|
| Iniciar | `docker compose up -d` |
| Ver estado | `docker compose ps` |
| Ver logs de Odoo | `docker compose logs -f odoo` |
| Reiniciar Odoo (p. ej. tras cambiar `odoo.conf` o agregar un módulo) | `docker compose restart odoo` |
| Detener sin borrar datos | `docker compose down` |
| Abrir `psql` | `docker compose exec db psql -U odoo -d odoo_docker` |
| Actualizar las imágenes | `docker compose pull && docker compose up -d` |

> ⚠️ `docker compose down -v` **elimina los volúmenes** y, con ellos, todas las bases de datos y los adjuntos.

## Módulos propios

1. Copia la carpeta del módulo (la que contiene `__manifest__.py`) dentro de `addons/`.
2. Reinicia Odoo: `docker compose restart odoo`.
3. En Odoo, activa el **modo desarrollador**, entra en **Aplicaciones → Actualizar lista de aplicaciones**, busca el módulo e instálalo.

Para actualizar un módulo ya instalado desde la terminal:

```bash
docker compose run --rm odoo odoo -d odoo_docker -u nombre_modulo --stop-after-init
docker compose restart odoo
```

## Respaldo y restauración

**Opción 1: desde el navegador.** Entra en http://localhost:8010/web/database/manager, usa **Backup** (formato *zip*, que incluye el filestore) y luego **Restore**. Ambas acciones piden la contraseña maestra.

**Opción 2: desde la terminal** (PowerShell o Bash):

```bash
# Base de datos
docker compose exec -T db pg_dump -U odoo -Fc odoo_docker > odoo_docker.dump

# Filestore (adjuntos)
docker compose cp odoo:/var/lib/odoo/.local/share/Odoo/filestore/odoo_docker ./filestore_odoo_docker
```

## Solución de problemas

| Síntoma | Causa probable | Qué hacer |
|---|---|---|
| `port is already allocated` al iniciar | Otro proceso usa el 8010 | Cambia el puerto del anfitrión en `ports` (p. ej. `"8011:8069"`) |
| Odoo muestra error de conexión a la base justo después de iniciar | Odoo arrancó antes de que PostgreSQL aceptara conexiones | `docker compose restart odoo` (o aplica la primera mejora sugerida) |
| El módulo no aparece en Aplicaciones | No se actualizó la lista o falta `__manifest__.py` | Revisa la estructura, reinicia Odoo y usa **Actualizar lista de aplicaciones** |
| Los cambios en `odoo.conf` no se aplican | Odoo solo lee la configuración al arrancar | `docker compose restart odoo` |

## Mejoras sugeridas

- **Esperar a que la base esté lista.** Ahora `depends_on` solo espera a que el contenedor `db` arranque, no a que pase el `healthcheck`. Con esta forma, Odoo espera a que PostgreSQL esté sano:
  ```yaml
  depends_on:
    db:
      condition: service_healthy
  ```
- **Sacar las credenciales del `docker-compose.yml`.** Mueve contraseñas como `odoo`/`odoo` a un archivo `.env` y no lo subas al repositorio.
- **Contraseña maestra fuerte.** Cambia `admin_passwd` antes de exponer el servidor fuera de `localhost`. Para que Odoo la guarde cifrada, cámbiala desde el gestor de bases de datos.
- **Fijar la versión exacta de la imagen** (p. ej. `odoo:19.0-AAAAMMDD`) para que `docker compose pull` no cambie la versión sin aviso.
- **Producción:** pon un proxy inverso con HTTPS (Nginx o Caddy) delante de Odoo, activa `proxy_mode = True` y define `list_db = False` en `odoo.conf`.
