# robot-2d-interfaz-web

Interfaz de gestión de la [carrera de robots autónomos](https://github.com/ojgarciab/carrera-robots-autonomos): usuarios, mundos, accesos y tokens de API.

> **Estado:** en diseño. Todavía no hay código; el plan de implementación está en [`Plan.md`](Plan.md).

## Qué es

Es la **aplicación web de administración** del sistema. Cada persona **inicia sesión con su cuenta** y, según su rol, gestiona sus tokens de API o administra usuarios y mundos.

Es además la **dueña del esquema de la base de datos** y de sus migraciones. La [pasarela](https://github.com/ghCreaR/robot-2d-pasarela) lee esa base de datos para validar tokens y accesos, pero no la crea ni la modifica (salvo la hora del último latido de cada mundo).

**No participa en los datos en tiempo real:** no se conecta al bus de mensajes ni a los motores de simulación. La sesión de esta web solo sirve para ella; los clientes de control y el visor usan tokens de API.

```
  navegador ──► INTERFAZ DE GESTIÓN ──► base de datos ◄── pasarela
  (sesión)       (este repo)            (PostgreSQL)       (lee y escucha NOTIFY)
                     │
                     └── NOTIFY: token revocado, acceso retirado…
```

## Funciones

### Para todos los usuarios

- **Ver sus mundos:** nombre, robots permitidos y si cada uno está **activo** ahora mismo.
- **Gestionar sus tokens de API:**
  - Generar un token de **lectura-escritura** (`crt_rw_…`) para su cliente de control o de **solo lectura** (`crt_ro_…`) para que otros vean sus sensores.
  - Elegir la caducidad: como mucho 7 días (`TOKEN_API_MAX_DIAS`).
  - El token se muestra **una sola vez** al generarlo; solo se guarda su *hash*.
  - Hay como máximo un token de cada tipo: generar uno nuevo revoca el anterior.
  - Revocar un token en cualquier momento.
- **Cambiar su contraseña** cuando quiera.

### Para los administradores

- **Usuarios:** darlos de alta, asignarles el rol (administrador o usuario), desactivarlos y **restablecer su contraseña**. No hay correo electrónico: el administrador comunica la contraseña al usuario, que puede cambiarla después.
- **Mundos:** darlos de alta con su nombre y sus robots permitidos (a partir de los modelos de `ROBOTS_DIR`), y **generar, regenerar o revocar su testigo de acceso**, que también se muestra una sola vez.
- **Accesos:** decidir a qué mundos tiene acceso cada usuario. Un usuario nuevo no tiene acceso a ninguno.

Cuando un cambio afecta a la pasarela (un token revocado, un acceso retirado, un usuario desactivado o un testigo revocado), lo avisa con **`NOTIFY` de PostgreSQL** para que surta efecto al instante.

## Configuración

Variables de entorno (ver el [`compose.yaml`](https://github.com/ojgarciab/carrera-robots-autonomos/blob/master/compose.yaml) del repositorio común):

| Variable | Por defecto | Descripción |
|----------|-------------|-------------|
| `DATABASE_URL` | — | Conexión a PostgreSQL. |
| `SESION_WEB_TTL` | `12h` | Duración de la sesión de esta web. No afecta a los tokens de API. |
| `TOKEN_API_MAX_DIAS` | `7` | Caducidad máxima de los tokens de API. |
| `ROBOTS_DIR` | — | Directorio con las definiciones de los robots, para elegir los permitidos en cada mundo. |
| `ADMIN_USUARIO`, `ADMIN_PASSWORD` | *(vacías)* | Si están definidas y no hay ningún administrador, se crea al arrancar. También se puede crear con `python manage.py crear_admin`. |

Escucha en el puerto `8080` del contenedor (publicado en el `8082` en el despliegue local).

## Documentación relacionada

- [Interfaz de gestión](https://github.com/ojgarciab/carrera-robots-autonomos#interfaz-de-gestión), [tokens de API](https://github.com/ojgarciab/carrera-robots-autonomos#tokens-de-api) y [registro de mundos](https://github.com/ojgarciab/carrera-robots-autonomos#registro-de-mundos)
- [Acceso de los usuarios a los mundos](https://github.com/ojgarciab/carrera-robots-autonomos#acceso-de-los-usuarios-a-los-mundos)
- [Contrato de la base de datos](https://github.com/ojgarciab/carrera-robots-autonomos/blob/master/contratos/base-de-datos.md): tablas compartidas con la pasarela y avisos `NOTIFY`
- [Plan de implementación](Plan.md)

## Licencia

[GPL-3.0](LICENSE).
