# Contrato de API

Contrato HTTP entre la SPA de React y la API de Spring WebFlux. Este documento
es la referencia común de los dos repositorios: si el código se desvía de aquí,
se corrige el código o se actualiza este documento de forma explícita.

Es una plantilla viva: la tabla de endpoints crece con cada funcionalidad
especificada.

---

## 1. Convenciones

### 1.1 Versionado

Todas las rutas cuelgan del prefijo `/api/v1`.

- Añadir un campo **opcional** a una respuesta, o un endpoint nuevo, no rompe
  el contrato y **no** sube la versión.
- Quitar o renombrar un campo, cambiar su tipo, o volver obligatorio un campo
  que era opcional **sí** rompe: eso obliga a `/api/v2`, con las dos versiones
  conviviendo hasta que el frontend migre.

### 1.2 Formato de las peticiones y respuestas

- `Content-Type: application/json; charset=utf-8` en peticiones con cuerpo.
- Nombres de campo en `camelCase`.
- **Fechas y horas:** ISO-8601 en UTC (`2026-09-05T14:30:00Z`).
- **Duraciones:** número entero de segundos (nunca cadenas como `"15m"`).
- **Identificadores:** UUID v4 en formato canónico.

### 1.3 Formato de errores

Todo error sigue **RFC 7807 (Problem Details)**, que Spring Boot 4 soporta de
forma nativa mediante `ProblemDetail`, con la cabecera
`Content-Type: application/problem+json` y una extensión propia `code`:

```json
{
  "type": "https://gestor-accesos/errors/invalid-credentials",
  "title": "Credenciales inválidas",
  "status": 401,
  "detail": "El email o la contraseña no son correctos.",
  "instance": "/api/v1/auth/login",
  "code": "INVALID_CREDENTIALS"
}
```

Reglas:

- **El frontend enruta por `code`, nunca por `detail`.** `detail` es texto para
  humanos y puede cambiar sin previo aviso; `code` es parte del contrato.
- Un error puede añadir campos extra al objeto raíz. Por ejemplo,
  `ACCOUNT_LOCKED` añade `retryAfterSeconds`.
- Los errores de validación añaden `errors`, un array de
  `{ "field": "email", "message": "..." }`.
- Los mensajes nunca revelan si un email está registrado, ni detalles internos
  (trazas, SQL, nombres de clase).

### 1.4 Autenticación

Dos credenciales, con responsabilidades separadas:

| Credencial | Dónde viaja | Vida | Para qué |
| --- | --- | --- | --- |
| **Access token** (JWT, **RS256**) | Cabecera `Authorization: Bearer <token>` | 900 s (15 min) | Autorizar cada petición |
| **Refresh token** (opaco) | Cookie `refresh_token` | 30 días | Obtener un access token nuevo |

La cookie de refresh se emite siempre con
`HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth`. **El refresh token
nunca viaja en el cuerpo ni en una cabecera**, y el frontend nunca puede leerlo
desde JavaScript.

Claims del access token, **todos parte del contrato**:

| Claim | Contenido |
| --- | --- |
| `sub` | UUID del usuario |
| `email` | Correo del usuario |
| `roles` | Códigos de rol, p. ej. `["BUYER","SELLER"]` |
| `permissions` | Permisos efectivos: unión de los roles, sin duplicados |
| `type` | `access`, `refresh` o `service` |
| `iss` | Emisor: este módulo |
| `aud` | Audiencia: el marketplace |
| `iat` / `exp` | Emisión y expiración |
| `jti` | Identificador del token |

El access token **no es revocable**: se confía en su vida corta.

**Se firma con RS256, no con HS256.** La firma asimétrica permite que los otros
seis módulos del marketplace verifiquen el token con la clave pública —publicada
en el JWKS del apartado 2.9— sin conocer ningún secreto nuestro y sin llamarnos
en cada petición. Con HS256 habría que repartir la clave de firma, y cualquiera
de los seis equipos podría emitir tokens haciéndose pasar por un administrador.

El coste está registrado en [`integracion.md` §5.1](integracion.md): un token
sigue siendo válido hasta su `exp` aunque el usuario haya sido desactivado.

### 1.5 Autenticación de servicio

Los otros módulos del marketplace no usan credenciales de usuario: cada uno
tiene un `client_id` y un `client_secret` propios y pide su token con
`POST /auth/token` (apartado 2.9). El token resultante lleva `type: "service"`,
el `client_id` en `sub` y la lista de `scopes` concedidos a ese módulo.

Un endpoint marcado como `Servicio` exige ese token y un scope concreto:
responde `401 TOKEN_INVALID` si falta o no es válido, y
`403 INSUFFICIENT_SCOPE` si el scope no alcanza. Los scopes están definidos en
[`integracion.md` §5.5](integracion.md).

Credenciales por módulo, y no una clave compartida, para que la auditoría diga
qué módulo consultó qué dato y para poder revocar uno sin afectar a los demás.

Un endpoint marcado como `Pública` no exige credenciales. Uno marcado como
`Bearer` responde `401` si falta o es inválido el access token.

### 1.6 Paginación

Aún no hay endpoints que devuelvan colecciones. Cuando los haya, se fija aquí
la convención (parámetros, forma de la respuesta) antes de implementarlos.

---

## 2. Endpoints

### 2.1 Tabla

| Método | Ruta | Auth | Descripción | Éxito | Spec |
| --- | --- | --- | --- | --- | --- |
| POST | `/api/v1/auth/login` | Pública | Autentica email + contraseña; emite access token y cookie de refresh | `200` | [login.md](login.md) |
| POST | `/api/v1/auth/refresh` | Cookie refresh | Renueva el access token y rota el refresh token | `200` | [login.md](login.md) |
| POST | `/api/v1/auth/logout` | Bearer | Revoca el refresh token y borra la cookie | `204` | [login.md](login.md) |
| GET | `/api/v1/auth/me` | Bearer | Devuelve el usuario de la sesión actual | `200` | [login.md](login.md) |
| POST | `/api/v1/auth/register` | Pública | Da de alta una cuenta y envía el correo de verificación | `202` | [registro.md](registro.md) |
| POST | `/api/v1/auth/verify-email` | Pública | Consume el token de verificación y marca el email como verificado | `200` | [registro.md](registro.md) |
| POST | `/api/v1/auth/resend-verification` | Pública | Reenvía el correo de verificación | `202` | [registro.md](registro.md) |
| GET | `/api/v1/.well-known/jwks.json` | Pública | Clave pública de verificación de firma | `200` | [integracion.md](integracion.md) |
| POST | `/api/v1/auth/token` | `client_secret` | Emite un token de servicio para un módulo consumidor | `200` | [integracion.md](integracion.md) |
| POST | `/api/v1/auth/introspect` | Servicio | Estado actual de un token de usuario | `200` | [integracion.md](integracion.md) |
| GET | `/api/v1/users/{id}` | Servicio | Datos básicos de un usuario | `200` | [integracion.md](integracion.md) |
| POST | `/api/v1/users/batch` | Servicio | Datos básicos de hasta 100 usuarios | `200` | [integracion.md](integracion.md) |
| GET | `/api/v1/users/{id}/addresses` | Servicio | Direcciones de entrega del cliente | `200` | [integracion.md](integracion.md) |
| GET | `/api/v1/roles` | Servicio | Catálogo de roles y sus permisos | `200` | [integracion.md](integracion.md) |

### 2.2 `POST /api/v1/auth/login`

Autentica a un usuario ya registrado.

**Petición**

```json
{
  "email": "ada@example.com",
  "password": "correct horse battery staple"
}
```

| Campo | Tipo | Reglas |
| --- | --- | --- |
| `email` | string | Obligatorio. Formato de email válido. Se normaliza a minúsculas y se recortan los espacios de los extremos. Máx. 254 caracteres. |
| `password` | string | Obligatorio. Entre 1 y 72 **bytes** UTF-8 (límite de BCrypt). No se normaliza nunca. |

**Respuesta `200 OK`**

```
Set-Cookie: refresh_token=<opaco>; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth; Max-Age=2592000
```

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
  "tokenType": "Bearer",
  "expiresIn": 900,
  "user": {
    "id": "9f1c2f4e-3a7b-4c8d-9e0f-1a2b3c4d5e6f",
    "email": "ada@example.com",
    "displayName": "Ada Lovelace",
    "roles": ["BUYER"]
  }
}
```

**Errores:** `VALIDATION_ERROR` (400), `INVALID_CREDENTIALS` (401),
`EMAIL_NOT_VERIFIED` (403), `ACCOUNT_DISABLED` (403), `ACCOUNT_LOCKED` (423).

### 2.3 `POST /api/v1/auth/refresh`

Renueva el access token. Sin cuerpo de petición: la credencial es la cookie.

**Respuesta `200 OK`** — mismo cuerpo que `/login`, y una cookie
`refresh_token` **nueva**: cada uso rota el token y revoca el anterior.

**Errores:** `TOKEN_EXPIRED` (401), `TOKEN_INVALID` (401) — incluye la cookie
ausente, el token ya rotado y el token revocado por un logout.

### 2.4 `POST /api/v1/auth/logout`

Revoca el refresh token de la sesión actual y borra la cookie
(`Max-Age=0`).

**Respuesta `204 No Content`**, sin cuerpo. Es **idempotente**: llamarlo con una
sesión ya cerrada también devuelve `204`.

El access token que el cliente tenga en memoria sigue siendo válido hasta su
`exp`; es responsabilidad del frontend descartarlo.

### 2.5 `GET /api/v1/auth/me`

Devuelve el usuario asociado al access token. La SPA lo usa para rehidratar la
sesión tras recargar la página.

**Respuesta `200 OK`** — el mismo objeto `user` que devuelve `/login`.

**Errores:** `TOKEN_EXPIRED` (401), `TOKEN_INVALID` (401).

---

### 2.6 `POST /api/v1/auth/register`

Da de alta una cuenta y encarga el envío del correo de verificación.

**Petición**

```json
{
  "email": "ada@example.com",
  "password": "una frase larga y poco comun",
  "displayName": "Ada Lovelace"
}
```

| Campo | Tipo | Reglas |
| --- | --- | --- |
| `email` | string | Obligatorio. Formato válido, máx. 254 caracteres. Se normaliza a minúsculas y se recortan los espacios de los extremos. |
| `password` | string | Obligatorio. Mínimo 12 caracteres y máximo 72 **bytes** UTF-8. Sin reglas de composición. No puede ser una contraseña común ni derivarse del email o del nombre. |
| `displayName` | string | Obligatorio. Entre 2 y 100 caracteres. Se normaliza a NFC; se rechazan los caracteres de control. |

**Respuesta `202 Accepted`**

```json
{
  "message": "Si la dirección es válida, recibirás un correo para confirmar tu cuenta."
}
```

> **La respuesta es idéntica exista o no la cuenta.** No se devuelve `409` ni
> ningún otro código que revele que el email ya está registrado: sería una vía
> de enumeración que echaría por tierra las defensas de `/auth/login`. Lo que
> cambia es el correo que recibe el titular. Ver
> [`registro.md` escenario 2](registro.md).

**Errores:** `VALIDATION_ERROR` (400), `WEAK_PASSWORD` (400).

### 2.7 `POST /api/v1/auth/verify-email`

Consume el token de verificación que viaja en el enlace del correo.

**Petición**

```json
{ "token": "<valor opaco de 256 bits>" }
```

**Respuesta `200 OK`** — el mismo objeto `user` que devuelve `/login`, ya con el
email verificado.

**Errores:** `VALIDATION_ERROR` (400), `VERIFICATION_TOKEN_INVALID` (410),
`VERIFICATION_TOKEN_EXPIRED` (410).

### 2.8 `POST /api/v1/auth/resend-verification`

Reemite el correo de verificación e invalida el token anterior.

**Petición**

```json
{ "email": "ada@example.com" }
```

**Respuesta `202 Accepted`** — mismo cuerpo genérico que `/auth/register`, y
también **idéntico** tanto si la cuenta no existe como si ya está verificada.

**Errores:** `VALIDATION_ERROR` (400), `RESEND_TOO_SOON` (429, añade
`retryAfterSeconds`).

---

Los apartados siguientes son la **API de integración**: la consumen los otros
seis módulos del marketplace, no nuestra SPA. Su spec es
[`integracion.md`](integracion.md).

### 2.9 `GET /api/v1/.well-known/jwks.json`

Publica la clave pública con la que cualquier módulo verifica la firma de
nuestros tokens sin llamarnos.

**Respuesta `200 OK`**

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "kid": "2026-09",
      "n": "0vx7agoebGcQSuu...",
      "e": "AQAB"
    }
  ]
}
```

Cacheable. Durante una rotación se publican **dos** claves y la anterior se
mantiene al menos 24 horas; el `kid` de la cabecera del token dice cuál usar.
Un consumidor que descargue este documento en cada validación anula la ventaja
de validar en local.

### 2.10 `POST /api/v1/auth/token`

Emite el token de servicio de un módulo consumidor.

**Petición** — `Content-Type: application/x-www-form-urlencoded`

```
grant_type=client_credentials&client_id=dispatch-module&client_secret=<secreto>
```

**Respuesta `200 OK`**

```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjIwMjYtMDkifQ...",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "scopes": ["users:read", "addresses:read"]
}
```

Los `scopes` son los concedidos a ese cliente, aunque haya pedido más.

**Errores:** `INVALID_CLIENT` (401), `VALIDATION_ERROR` (400).

### 2.11 `POST /api/v1/auth/introspect`

Estado **actual** de un token de usuario. Scope: `tokens:introspect`.

**Petición**

```json
{ "token": "eyJhbGciOiJSUzI1NiJ9..." }
```

**Respuesta `200 OK`** — token utilizable

```json
{
  "active": true,
  "sub": "9f1c2f4e-3a7b-4c8d-9e0f-1a2b3c4d5e6f",
  "email": "ada@example.com",
  "roles": ["BUYER"],
  "permissions": [],
  "expiresAt": "2026-09-05T14:45:00Z"
}
```

**Respuesta `200 OK`** — token no utilizable

```json
{ "active": false, "reason": "USER_INACTIVE" }
```

`reason` toma uno de `USER_INACTIVE`, `USER_LOCKED`, `TOKEN_EXPIRED` o
`TOKEN_INVALID`. Un token mal firmado y uno de un usuario desactivado devuelven
los dos `200` con `active: false`: **la introspección no falla, informa**.

Los roles y permisos que devuelve son los **vigentes ahora**, no los que llevaba
el token.

**Errores:** `TOKEN_INVALID` (401), `INSUFFICIENT_SCOPE` (403).

### 2.12 `GET /api/v1/users/{id}`

Datos básicos de un usuario. Scope: `users:read`.

**Respuesta `200 OK`**

```json
{
  "id": "9f1c2f4e-3a7b-4c8d-9e0f-1a2b3c4d5e6f",
  "displayName": "Ada Lovelace",
  "email": "ada@example.com",
  "status": "ACTIVE",
  "emailVerified": true,
  "roles": ["BUYER"]
}
```

Nunca incluye el hash de la contraseña. El número de documento solo aparece con
scope `users:read:document`, y sin él llega enmascarado a los últimos tres
dígitos.

**Errores:** `TOKEN_INVALID` (401), `INSUFFICIENT_SCOPE` (403), `NOT_FOUND`
(404).

### 2.13 `POST /api/v1/users/batch`

Hasta 100 usuarios en una llamada. Scope: `users:read`.

**Petición**

```json
{ "ids": ["9f1c2f4e-...", "3b2d1a0c-...", "7e5f4c3b-..."] }
```

**Respuesta `200 OK`**

```json
{
  "users": [
    { "id": "9f1c2f4e-...", "displayName": "Ada Lovelace", "email": "ada@example.com", "status": "ACTIVE", "emailVerified": true, "roles": ["BUYER"] }
  ],
  "notFound": ["3b2d1a0c-...", "7e5f4c3b-..."]
}
```

**Los identificadores no hallados no fallan la petición:** van a `notFound` y el
resto se devuelve. Los duplicados se colapsan.

**Errores:** `VALIDATION_ERROR` (400, algún `id` no es un UUID),
`BATCH_TOO_LARGE` (400, más de 100), `TOKEN_INVALID` (401),
`INSUFFICIENT_SCOPE` (403).

### 2.14 `GET /api/v1/users/{id}/addresses` y `GET /api/v1/roles`

`/addresses` devuelve las direcciones de entrega del cliente con scope
`addresses:read`, **sin** documento ni fecha de nacimiento. `/roles` devuelve el
catálogo de roles con sus permisos con scope `roles:read`.

La forma exacta de sus cuerpos depende de specs todavía no escritas —atributos
de usuario y catálogo de roles—, así que se fija al aprobarlas. Lo que sí es
contrato desde ya son la ruta, el scope y los códigos de error.

## 3. Catálogo de códigos de error

| `code` | HTTP | Cuándo se devuelve |
| --- | --- | --- |
| `VALIDATION_ERROR` | 400 | El cuerpo no cumple las reglas de los campos (email malformado, campo vacío, contraseña de más de 72 bytes) |
| `INVALID_CREDENTIALS` | 401 | Email no registrado **o** contraseña incorrecta. Idéntico en ambos casos, a propósito |
| `TOKEN_EXPIRED` | 401 | El access o el refresh token superó su `exp` |
| `TOKEN_INVALID` | 401 | Token ausente, malformado, con firma inválida, ya rotado o revocado |
| `EMAIL_NOT_VERIFIED` | 403 | Credenciales correctas, pero la cuenta no ha verificado su email |
| `ACCOUNT_DISABLED` | 403 | Cuenta desactivada por un administrador |
| `ACCOUNT_LOCKED` | 423 | Bloqueo temporal por intentos fallidos. Añade `retryAfterSeconds` |
| `WEAK_PASSWORD` | 400 | Contraseña más corta que el mínimo, demasiado común, de más de 72 bytes, o derivada del email o del nombre |
| `VERIFICATION_TOKEN_INVALID` | 410 | Token de verificación inexistente, malformado o ya consumido |
| `VERIFICATION_TOKEN_EXPIRED` | 410 | Token de verificación emitido hace más de 24 h |
| `RESEND_TOO_SOON` | 429 | Reenvío pedido antes de que pase el cooldown de 60 s. Añade `retryAfterSeconds` |
| `INVALID_CLIENT` | 401 | `client_id` o `client_secret` de servicio incorrectos. Idéntico en ambos casos, a propósito |
| `INSUFFICIENT_SCOPE` | 403 | Token de servicio válido, pero sin el scope que el endpoint exige. No revela cuál haría falta |
| `BATCH_TOO_LARGE` | 400 | Consulta por lote con más de 100 identificadores. No se trunca en silencio |
| `NOT_FOUND` | 404 | El recurso consultado no existe. Solo se llega aquí con un token de servicio válido y el scope correcto |

Los códigos son estables: uno publicado no se renombra ni cambia de significado
sin subir la versión de la API.

---

## 4. Endpoints pendientes de especificar

De las funcionalidades que `overview.md` declara dentro del alcance, quedan sin
contrato:

| Funcionalidad | Endpoints previsibles |
| --- | --- |
| Recuperación de contraseña | `POST /auth/forgot-password`, `POST /auth/reset-password` |
| Cambio de contraseña autenticado | `POST /auth/change-password` |
| Roles y permisos | Cuerpo de `GET /roles` y `GET /permissions`; la ruta y el scope de `/roles` ya están fijados en 2.14 |
| Atributos de usuario y direcciones | Cuerpo de `GET /users/{id}/addresses`; su ruta y su scope ya están fijados en 2.14 |
| Administración de cuentas | `POST /admin/users/{id}/disable`, `/enable`, `/unlock` |
| Eventos asíncronos entre módulos | Contratos de mensaje de `user.deactivated`, `user.locked`, `user.roles_changed`, `user.created` y `user.attributes_updated`. Es la vía 3 de [`integracion.md`](integracion.md) |

Son una previsión, no un compromiso: cada funcionalidad nueva añade sus filas a
la tabla de la sección 2 y sus códigos a la sección 3 **después** de aprobar su
spec, y las rutas pueden cambiar al escribirla.
