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
| **Access token** (JWT, HS256) | Cabecera `Authorization: Bearer <token>` | 900 s (15 min) | Autorizar cada petición |
| **Refresh token** (opaco) | Cookie `refresh_token` | 30 días | Obtener un access token nuevo |

La cookie de refresh se emite siempre con
`HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth`. **El refresh token
nunca viaja en el cuerpo ni en una cabecera**, y el frontend nunca puede leerlo
desde JavaScript.

Claims del access token: `sub` (id de usuario), `email`, `roles`, `iat`, `exp`,
`jti`. El access token **no es revocable**: se confía en su vida corta.

Un endpoint marcado como `Pública` no exige credenciales. Uno marcado como
`Bearer` responde `401` si falta o es inválido el access token.

### 1.5 Paginación

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

Los códigos son estables: uno publicado no se renombra ni cambia de significado
sin subir la versión de la API.

---

## 4. Endpoints pendientes de especificar

Ninguno todavía. Cada funcionalidad nueva añade sus filas a la tabla de la
sección 2 y sus códigos a la sección 3, siempre después de aprobar su spec.
