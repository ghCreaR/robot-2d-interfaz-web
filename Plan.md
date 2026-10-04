# Plan de implementación · robot-2d-interfaz-web

Este plan detalla cómo construir la interfaz de gestión descrita en el [README del repositorio común](https://github.com/ojgarciab/carrera-robots-autonomos). Todavía no hay código: es una propuesta para revisar antes de empezar.

## 1. Decisiones técnicas propuestas

| Tema | Propuesta | Motivo |
|------|-----------|--------|
| Lenguaje y *framework* | **Python 3.12** con **Django 5** | Trae casi todo lo que pide este componente: sesiones, autenticación, formularios con CSRF, migraciones del esquema y plantillas. Coherente con el `.gitignore` del repositorio. |
| Base de datos | **PostgreSQL** con `psycopg` 3 | La del despliegue de referencia. |
| Contraseñas | **Argon2** (`argon2-cffi`) | Algoritmo recomendado; Django lo admite de serie. |
| Interfaz | Plantillas de Django con **HTML y CSS sencillos**, sin compilación de JavaScript | Es una herramienta de administración con pocas pantallas. **htmx** opcional para acciones sin recargar. |
| Servidor | **Gunicorn** detrás del puerto `8080`; ficheros estáticos con **WhiteNoise** | Sin necesidad de un servidor web aparte. |
| Pruebas | **pytest-django** | |
| Calidad | **ruff** y **mypy** (`django-stubs`) | |

Alternativa considerada: FastAPI con SQLAlchemy y Alembic. Habría que construir a mano las sesiones, los formularios, el CSRF y la administración, que Django ya resuelve.

## 2. Estructura del repositorio

```
robot-2d-interfaz-web/
├── pyproject.toml
├── Dockerfile
├── manage.py
├── gestion/                    # proyecto Django: settings, urls, wsgi
└── apps/
    ├── cuentas/                # usuarios, roles, inicio de sesión, contraseña
    ├── tokens/                 # tokens de API: generar, listar, revocar
    ├── mundos/                 # mundos, testigos, robots permitidos, estado activo
    ├── accesos/                # acceso de los usuarios a los mundos
    └── avisos/                 # envío de NOTIFY a la pasarela
```

## 3. Esquema de la base de datos

Las tablas que lee la pasarela llevan **nombres fijos** (`db_table`) y forman parte del contrato, así que no dependen de los nombres internos de Django. Cualquier cambio en ellas debe coordinarse con la pasarela.

| Tabla | Campos principales | Notas |
|-------|--------------------|-------|
| `usuarios` | `id` (UUID), `nombre_usuario` (único), `nombre_visible`, `password` (hash Argon2), `rol` (`admin` \| `usuario`), `activo`, `creado` | Modelo de usuario propio de Django (`AUTH_USER_MODEL`), creado desde la primera migración. El `nombre_visible` es la etiqueta que se ve sobre el robot en la vista de administrador. |
| `tokens_api` | `id`, `usuario_id`, `tipo` (`rw` \| `ro`), `hash` (SHA-256, único), `pista` (últimos caracteres, para reconocerlo), `caduca`, `revocado`, `creado`, `ultimo_uso` | Como mucho un token vivo de cada tipo por usuario, garantizado con un índice único parcial (`WHERE NOT revocado`). |
| `mundos` | `id` (UUID), `nombre`, `robots_permitidos` (array de texto), `testigo_hash`, `testigo_creado`, `ultimo_latido`, `creado` | `ultimo_latido` lo escribe la pasarela; un mundo se considera activo si es reciente (por ejemplo, de menos de 15 s). |
| `accesos` | `usuario_id`, `mundo_id`, `concedido_por`, `creado` | Clave primaria (`usuario_id`, `mundo_id`). |
| `historial_carreras` | *(fase posterior)* | El README común lo menciona, pero todavía no hay requisitos. |

**Tokens y testigos.** Se generan con `secrets.token_urlsafe(32)` (256 bits). Los tokens llevan el prefijo de su tipo (`crt_rw_` / `crt_ro_`). Como son aleatorios y largos, basta con guardar su SHA-256: no hace falta un *hash* lento como el de las contraseñas, y así la pasarela puede validarlos rápido.

### 3.1. Avisos a la pasarela (`NOTIFY`)

Se envían **dentro de la misma transacción** que el cambio, con `transaction.on_commit` o un *trigger* de PostgreSQL, para que la pasarela nunca reciba el aviso de algo que luego se deshace.

| Canal | Carga | Cuándo |
|-------|-------|--------|
| `token_revocado` | hash del token | Al revocar un token, o al generar otro del mismo tipo. |
| `acceso_retirado` | `usuario_id`, `mundo_id` | Al quitar el acceso de un usuario a un mundo. |
| `usuario_desactivado` | `usuario_id` | Al desactivar un usuario. Invalida todos sus tokens. |
| `testigo_revocado` | `mundo_id` | Al regenerar o revocar el testigo de un mundo. |

Se propone hacerlo con **triggers de PostgreSQL** creados en las migraciones: así el aviso sale aunque el cambio se haga por otra vía (por ejemplo, a mano en la base de datos).

## 4. Pantallas

| Ruta | Quién | Contenido |
|------|-------|-----------|
| `/entrar`, `/salir` | todos | Inicio y cierre de sesión. |
| `/` | todos | Mis mundos: nombre, robots permitidos y si está activo. |
| `/tokens` | todos | Mis tokens: tipo, pista, caducidad y último uso; generar (con caducidad ≤ `TOKEN_API_MAX_DIAS`) y revocar. El token nuevo se muestra una sola vez, con un botón para copiarlo y un aviso claro. |
| `/cuenta` | todos | Cambiar la contraseña. |
| `/admin/usuarios` | administradores | Listado, alta, rol, activar o desactivar y restablecer la contraseña. |
| `/admin/mundos` | administradores | Listado con su estado, alta, edición del nombre y de los robots permitidos, y generar, regenerar o revocar el testigo. Al darlo de alta se muestran el UUID y el testigo listos para copiar a `.env` (`MUNDO_…_UUID` y `MUNDO_…_TESTIGO`). |
| `/admin/accesos` | administradores | Tabla de usuarios × mundos para conceder y retirar accesos. |

No se usa el `/admin` de Django para la gestión diaria: las reglas (un token de cada tipo, testigos que se muestran una sola vez, avisos `NOTIFY`) se aplican mejor en vistas propias. Se puede dejar activado solo para depurar.

## 5. Seguridad

- **Sesión:** cookie `HttpOnly`, `Secure` (configurable para el despliegue local por HTTP), `SameSite=Lax`, duración `SESION_WEB_TTL`. Se renueva el identificador al iniciar sesión.
- **CSRF** en todos los formularios (Django lo trae de serie).
- **Inicio de sesión:** límite de intentos por usuario y por IP (por ejemplo con `django-axes`) y mensajes de error que no revelan si el usuario existe.
- **Permisos:** un decorador o *mixin* exige el rol de administrador en todas las vistas `/admin/…`.
- **Secretos:** los tokens y los testigos no se guardan ni se registran nunca en claro.
- **Cabeceras:** CSP estricta, `X-Frame-Options: DENY` y `Referrer-Policy`.

## 6. Primer administrador

El README común pide "entrar como administrador" en la puesta en marcha, pero aún no dice cómo se crea. Se propone:

- Un comando `python manage.py crear_admin`, que se ejecuta con `docker compose exec interfaz …`.
- Además, de forma opcional, las variables `ADMIN_USUARIO` y `ADMIN_PASSWORD`: si están definidas y no hay ningún administrador, se crea al arrancar. Habría que añadirlas al `compose.yaml` y al `.env.example` del repositorio común.

## 7. Fases de implementación

### Fase 0 · Esqueleto
- Proyecto Django, `pyproject.toml`, `ruff`, `mypy`, `pytest-django` y GitHub Actions con un servicio `postgres`.
- `settings` desde variables de entorno (`DATABASE_URL`, `SESION_WEB_TTL`, `TOKEN_API_MAX_DIAS`, `ROBOTS_DIR`).
- `Dockerfile`: aplica las migraciones al arrancar y después lanza Gunicorn en el `8080`, con `HEALTHCHECK`.

### Fase 1 · Esquema y cuentas
- Modelo de usuario propio y migración inicial con todas las tablas de la sección 3.
- Inicio y cierre de sesión, cambio de contraseña, comando `crear_admin`.
- Publicar el esquema (por ejemplo, como SQL generado o documentación) en el repositorio común para la pasarela.

### Fase 2 · Tokens de API
- Generación con prefijo y *hash*, regla de uno por tipo (revoca el anterior), caducidad máxima, revocación y pantalla de "se muestra una sola vez".
- *Trigger* `token_revocado`.
- Pruebas: el token generado valida contra el *hash*; generar uno nuevo revoca el anterior; no se acepta una caducidad mayor que el máximo.

### Fase 3 · Mundos
- Alta y edición, lista de robots permitidos leída de `ROBOTS_DIR` (solo los `id` válidos), testigo generado una sola vez y su regeneración o revocación.
- Estado activo a partir de `ultimo_latido`.
- *Trigger* `testigo_revocado`.

### Fase 4 · Usuarios y accesos
- Gestión de usuarios por los administradores.
- Tabla de accesos.
- *Triggers* `acceso_retirado` y `usuario_desactivado`.
- "Mis mundos" filtrado por accesos.

### Fase 5 · Acabado
- Estilos sencillos y adaptables al móvil, mensajes claros y confirmación antes de revocar.
- Límite de intentos de inicio de sesión y cabeceras de seguridad.
- Prueba de extremo a extremo con la pasarela: un token revocado deja de funcionar en la pasarela al instante.

### Fase 6 · Posterior
- Historial de carreras, cuando se definan sus requisitos.

## 8. Dependencias con otros repositorios

| Depende de | Qué necesita |
|------------|--------------|
| Repositorio común | Directorio `robots/` (ya existe) y, si se aprueba, variables para el primer administrador en `compose.yaml`. |
| `robot-2d-pasarela` | Escribe `mundos.ultimo_latido` y escucha los `NOTIFY`. Los nombres de las tablas y canales de la sección 3 son el contrato con ella. |

## 9. Preguntas abiertas

1. **Primer administrador:** ¿comando, variables de entorno o las dos cosas?
2. **Registro de usuarios:** ¿solo los dan de alta los administradores, como dice el README común, o se quiere también una invitación por correo?
3. **Contraseñas olvidadas:** ¿basta con que un administrador las restablezca, o hace falta recuperación por correo (y por tanto configurar SMTP)?
4. **Mundo activo:** ¿cuántos segundos sin latido marcan un mundo como parado en esta web? Se propone 15 s.
