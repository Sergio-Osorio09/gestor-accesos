# Kit de integración para los equipos consumidores

Guía práctica para los otros seis módulos del marketplace. Si tu equipo necesita
saber **quién** está detrás de una petición y **qué se le permite hacer**, esto es
lo que hay que leer, y en este orden.

El contrato formal está en [`integracion.md`](integracion.md) y
[`api-contract.md`](api-contract.md); esto es el resumen operativo.

## 1. Lo que necesitas de nosotros

Pídenoslo en el canal de integración. Son seis cosas:

| Qué | Para qué |
| --- | --- |
| URL base de cada entorno | Simulado, desarrollo y nube |
| `client_id` y `client_secret` de tu módulo | Autenticarte cuando **tú** nos llamas |
| La lista de scopes concedidos a tu módulo | Saber qué endpoints puedes usar |
| Credenciales de usuarios de prueba | Uno por rol, para tus pruebas |
| El OpenAPI ([`openapi.yaml`](openapi.yaml)) | Generar tu cliente |
| La colección de Postman o Bruno | Probar a mano |

El `client_secret` viaja por canal privado, nunca por el grupo del curso ni en un
repositorio.

## 2. Empieza hoy, sin esperar a nuestra implementación

El contrato está congelado antes que el código. Levanta el entorno simulado y
programa contra él:

```bash
npx @stoplight/prism-cli mock specs/openapi.yaml -p 4010
```

Responde con los ejemplos reales del contrato. Para forzar un caso concreto usa
la cabecera `Prefer`:

```bash
# El camino feliz
curl http://localhost:4010/users/9f1c2f4e-3a7b-4c8d-9e0f-1a2b3c4d5e6f \
     -H 'Authorization: Bearer tu-token-de-servicio'

# Un 403 por scope insuficiente
curl -i http://localhost:4010/roles -H 'Prefer: code=403'

# Un usuario desactivado en la introspección
curl -X POST http://localhost:4010/auth/introspect \
     -H 'Content-Type: application/json' \
     -H 'Prefer: example=usuarioDesactivado' \
     -d '{"token":"eyJ..."}'
```

Cuando nuestra implementación esté lista, solo cambias la URL base.

## 3. Cómo verificas la identidad de un usuario

**Este es el 95 % de tu integración, y no nos llama.**

Cuando te llega una petición con `Authorization: Bearer <token>`:

1. Descarga **una vez** `GET /api/v1/.well-known/jwks.json` y **cachéalo**.
2. Lee el `kid` de la cabecera del token y elige esa clave del JWKS.
3. Verifica la firma (RS256), el `iss`, el `aud` y el `exp`.
4. Lee `sub`, `roles` y `permissions` del cuerpo del token.

No nos llames para esto. Si lo haces en cada petición, te conviertes en rehén de
nuestra disponibilidad: si nosotros caemos, cae tu módulo.

Ejemplo con Spring Boot, que es lo que usa la mayoría de los equipos:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: http://localhost:8080/api/v1/.well-known/jwks.json
          issuer-uri: https://gestor-accesos
```

Con eso Spring Security valida el token solo, cachea el JWKS y expone los claims.
No hace falta escribir el filtro a mano.

### Lo que llega dentro del token

| Claim | Contenido |
| --- | --- |
| `sub` | UUID del usuario |
| `email` | Correo |
| `roles` | `["BUYER"]`, `["SELLER","SALES_ADMIN"]`… |
| `permissions` | Permisos efectivos, sin duplicados |
| `type` | `access`, `refresh` o `service` |
| `iss` / `aud` | Emisor y audiencia |
| `iat` / `exp` | Emisión y expiración (900 s) |
| `jti` | Identificador del token |

**Autoriza por `roles` y `permissions`, nunca por `email`.** Un correo puede
cambiar; un `sub` no.

## 4. Cuándo sí tienes que llamarnos

Solo en dos situaciones.

### a) Antes de una operación sensible

Un token vive 15 minutos y **sigue siendo válido aunque hayamos desactivado o
bloqueado al usuario**. Es el precio de que puedas validar sin llamarnos.

Si vas a anular un pedido, hacer un reembolso o cambiar un precio, confirma el
estado actual:

```bash
curl -X POST $BASE/auth/introspect \
     -H "Authorization: Bearer $TU_TOKEN_DE_SERVICIO" \
     -H 'Content-Type: application/json' \
     -d '{"token":"<el token del usuario>"}'
```

```json
{ "active": false, "reason": "USER_INACTIVE" }
```

`active: false` significa **niega la operación**, aunque la firma fuera correcta.

### b) Para datos que el token no lleva

```bash
# Uno
GET  $BASE/users/{id}                 # scope users:read
# Muchos, hasta 100 de una vez
POST $BASE/users/batch                # scope users:read
# Direcciones de entrega
GET  $BASE/users/{id}/addresses       # scope addresses:read
# Catálogo de roles
GET  $BASE/roles                      # scope roles:read
```

En el lote, **los identificadores que no existen no rompen la petición**: llegan
en `notFound` y el resto se devuelve.

```json
{
  "users": [ { "id": "9f1c2f4e-...", "displayName": "Ada Lovelace", "...": "..." } ],
  "notFound": ["3b2d1a0c-...", "7e5f4c3b-..."]
}
```

## 5. Cómo te autenticas tú

Tu módulo tiene una identidad propia, distinta de la del usuario final:

```bash
curl -X POST $BASE/auth/token \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -d "grant_type=client_credentials&client_id=$TU_ID&client_secret=$TU_SECRETO"
```

```json
{ "accessToken": "eyJ...", "tokenType": "Bearer", "expiresIn": 3600,
  "scopes": ["users:read", "addresses:read"] }
```

Cachéalo hasta que le falte poco para caducar; no pidas uno por petición.

Los scopes que existen:

| Scope | Qué autoriza |
| --- | --- |
| `tokens:introspect` | Introspección de tokens de usuario |
| `users:read` | Datos básicos, individuales y por lote |
| `users:read:document` | Documento en claro en vez de enmascarado |
| `addresses:read` | Direcciones de entrega |
| `roles:read` | Catálogo de roles y permisos |

Si te falta uno, pídelo en la sincronización entre equipos explicando para qué.
No los concedemos sobre la marcha: cada consumidor recibe **solo** los campos que
su scope autoriza.

## 6. Errores: enruta por `code`, nunca por `detail`

Todos siguen RFC 7807 con la extensión `code`:

```json
{
  "type": "https://gestor-accesos/errors/insufficient-scope",
  "title": "Permiso insuficiente",
  "status": 403,
  "detail": "Este cliente no tiene permiso para esta operación.",
  "code": "INSUFFICIENT_SCOPE"
}
```

**`detail` es texto para humanos y puede cambiar sin aviso. `code` es contrato.**

| `code` | HTTP | Qué hacer |
| --- | --- | --- |
| `TOKEN_INVALID` | 401 | Tu token de servicio falta o no vale. Pide uno nuevo |
| `INVALID_CLIENT` | 401 | Tu `client_id` o `client_secret` están mal |
| `INSUFFICIENT_SCOPE` | 403 | Te falta un scope. Pídelo, no reintentes |
| `BATCH_TOO_LARGE` | 400 | Parte tu lote en trozos de 100 |
| `VALIDATION_ERROR` | 400 | Revisa el cuerpo; mira el array `errors` |
| `NOT_FOUND` | 404 | Ese usuario no existe |

## 7. Lo que **no** hacemos por ti

- **No autorizamos tus operaciones.** Te damos identidad, roles y permisos; qué
  permite hacer cada endpoint tuyo lo decides tú.
- **No te avisamos todavía por evento** de una baja o un bloqueo. Los eventos
  asíncronos son la vía 3 y aún no existen: hasta entonces, la ventana de desfase
  es de 15 minutos y quien no la tolere usa la introspección.
- **No te damos acceso a nuestra base de datos.** Ni lo pidas: la matriz de
  integración del curso lo prohíbe, y es lo que evita que un cambio de esquema
  nuestro rompa tu despliegue.
- **No emitimos tokens de usuario para ti.** Tu token de servicio identifica a tu
  módulo, no suplanta a una persona.

## 8. Si algo del contrato tiene que cambiar

No renombramos campos que ya consumes. Añadimos el nuevo, mantenemos el
anterior, avisamos en el canal de integración con **al menos una semana** de
antelación y retiramos el viejo solo cuando todos los módulos afectados
confirman.

Un cambio que no admita esa convivencia obliga a `/api/v2`, con las dos versiones
funcionando hasta que migres.

## 9. Pendiente de acuerdo entre equipos

Antes de que publiquemos el contrato como definitivo hay que cerrar esto con
vosotros:

- **Los códigos de rol.** Aquí están en inglés (`BUYER`, `SELLER`, `SALES_ADMIN`,
  `DISPATCH_MANAGER`, `COMMERCIAL_MANAGER`, `SYSTEM_ADMIN`); el plan del curso los
  define en español (`CLIENTE`, `VENDEDOR`, …). **Viajan dentro del token que
  vosotros leéis**, así que hay que elegir uno y que sea el mismo para los siete
  módulos.
- **El catálogo de permisos.** Los códigos tipo `ORDER_CANCEL` o `PRODUCT_CREATE`
  los define cada módulo dueño de esa operación. Mientras no exista el catálogo,
  `permissions` llega vacío y se autoriza por `roles`.
- **Qué campos necesita cada uno.** Decidnos qué consultáis de verdad para
  ajustar los scopes al mínimo necesario.
