# Login con email y contraseña

## 1. Contexto

Permite que un usuario ya registrado se autentique con su email y su contraseña
y obtenga una sesión válida. Es la puerta de entrada al resto del marketplace:
sin esta funcionalidad ninguna otra puede identificar a quien la usa.

## 2. Actores

| Actor | Descripción |
| --- | --- |
| **Usuario registrado** | Persona con una cuenta existente. Su cuenta tiene un estado (`ACTIVE` / `DISABLED`), un flag de verificación de email y, opcionalmente, un bloqueo temporal vigente. |
| **SPA (frontend React)** | Presenta el formulario, envía las credenciales, guarda el access token **en memoria** y renueva la sesión de forma transparente. |
| **API de autenticación (Spring WebFlux)** | Valida credenciales, aplica la política de bloqueo, emite y rota tokens. |
| **Administrador** | No participa en el flujo de login. Aparece solo como el actor que deja una cuenta en estado `DISABLED`; su gestión queda fuera de alcance (bloque 5). |

Los estados de cuenta y cómo se representan están definidos en
[`overview.md`](overview.md#estados-de-una-cuenta), que es la fuente de verdad.
Los que intervienen en esta funcionalidad:

- `emailVerified`: `true` / `false`.
- `status`: `ACTIVE` (operativa) o `DISABLED` (desactivada por un administrador).
- `lockedUntil`: instante hasta el que la cuenta está bloqueada por intentos
  fallidos, o vacío si no lo está.

Las cuentas las crea [`registro.md`](registro.md); esta spec asume que ya
existen.

## 3. Escenarios

### Escenario 1 — Login exitoso

**GIVEN** un usuario con cuenta `ACTIVE`, `emailVerified = true`, sin bloqueo
vigente y sin intentos fallidos acumulados
**WHEN** envía `POST /api/v1/auth/login` con su email y su contraseña correcta
**THEN** la respuesta es `200` con `accessToken`, `tokenType: "Bearer"`,
`expiresIn: 900` y el objeto `user` con su `id`, `email`, `displayName` y `roles`
**AND** la respuesta incluye una cookie `refresh_token` con
`HttpOnly`, `Secure`, `SameSite=Strict` y `Path=/api/v1/auth`
**AND** el contador de intentos fallidos de la cuenta queda en `0`.

### Escenario 2 — Contraseña incorrecta

**GIVEN** un usuario con cuenta `ACTIVE` y verificada
**WHEN** envía `POST /api/v1/auth/login` con su email correcto y una contraseña
incorrecta
**THEN** la respuesta es `401` con `code: INVALID_CREDENTIALS`
**AND** no se emite ningún token ni ninguna cookie
**AND** el contador de intentos fallidos de esa cuenta se incrementa en `1`.

### Escenario 3 — Email no registrado

**GIVEN** un email que no corresponde a ninguna cuenta
**WHEN** se envía `POST /api/v1/auth/login` con ese email y cualquier contraseña
**THEN** la respuesta es `401` con `code: INVALID_CREDENTIALS` — **exactamente
la misma** que la del escenario 2, en cuerpo y en tiempo de respuesta
**AND** no se crea ningún registro ni contador para ese email.

### Escenario 4 — Email registrado pero sin verificar

**GIVEN** un usuario con cuenta `ACTIVE`, `emailVerified = false` y sin bloqueo
vigente
**WHEN** envía `POST /api/v1/auth/login` con su **contraseña correcta**
**THEN** la respuesta es `403` con `code: EMAIL_NOT_VERIFIED`
**AND** no se emite ningún token
**AND** el contador de intentos fallidos **no** se incrementa: la credencial era
válida.

### Escenario 5 — Quinto intento fallido consecutivo: bloqueo

**GIVEN** un usuario verificado que acumula 4 intentos fallidos consecutivos
**WHEN** envía un quinto `POST /api/v1/auth/login` con contraseña incorrecta
**THEN** la respuesta es `423` con `code: ACCOUNT_LOCKED` y
`retryAfterSeconds: 900`
**AND** la cuenta queda bloqueada durante 15 minutos
**AND** durante esa ventana ni siquiera la contraseña correcta abre sesión.

### Escenario 6 — Intento sobre una cuenta bloqueada

**GIVEN** un usuario cuya cuenta tiene un bloqueo vigente
**WHEN** envía `POST /api/v1/auth/login`, con la contraseña correcta o incorrecta
**THEN** la respuesta es `423` con `code: ACCOUNT_LOCKED` y un
`retryAfterSeconds` igual al tiempo restante de bloqueo
**AND** el contador de intentos fallidos **no** se incrementa: un bloqueo no se
puede prolongar a base de reintentos.

### Escenario 7 — Renovación del access token

**GIVEN** una sesión abierta cuyo access token ha caducado, con una cookie
`refresh_token` válida y no rotada
**WHEN** la SPA envía `POST /api/v1/auth/refresh` con esa cookie
**THEN** la respuesta es `200` con un access token nuevo y `expiresIn: 900`
**AND** la respuesta trae una cookie `refresh_token` **nueva**
**AND** el refresh token anterior queda revocado y su reutilización se rechaza.

### Escenario 8 — Cierre de sesión

**GIVEN** una sesión abierta con un access token válido
**WHEN** el usuario envía `POST /api/v1/auth/logout`
**THEN** la respuesta es `204` sin cuerpo
**AND** la cookie `refresh_token` se borra (`Max-Age=0`)
**AND** ese refresh token queda revocado: un `POST /api/v1/auth/refresh`
posterior responde `401` con `code: TOKEN_INVALID`.

### Escenario 9 — Rehidratación de la sesión

**GIVEN** un usuario que recarga la página con un access token vigente
**WHEN** la SPA envía `GET /api/v1/auth/me` con `Authorization: Bearer <token>`
**THEN** la respuesta es `200` con el mismo objeto `user` que devolvió el login
**AND** la SPA restaura la sesión sin volver a pedir credenciales.

## 4. Casos límite

### 4.1 Credenciales inválidas

Un email inexistente y una contraseña incorrecta devuelven el mismo `code`
(`INVALID_CREDENTIALS`), el mismo cuerpo y **el mismo tiempo de respuesta**.
Cuando el email no existe se ejecuta igualmente un hash señuelo contra un valor
fijo, para que la duración de la petición no delate qué cuentas están
registradas.

### 4.2 Normalización del email

El email se compara en minúsculas y sin espacios en los extremos:
`" Ada@Example.com "` y `"ada@example.com"` son la misma cuenta. **La contraseña
no se normaliza nunca** — ni espacios, ni mayúsculas, ni forma Unicode.

### 4.3 Contraseña fuera de los límites

Una contraseña vacía, o de más de **72 bytes** UTF-8, se rechaza en validación
con `400 VALIDATION_ERROR`. El límite no es arbitrario: BCrypt trunca en
silencio a partir de ahí, y aceptarla daría una falsa sensación de seguridad.
Ojo, el límite es en **bytes**, no en caracteres: los acentos y emojis ocupan
más de uno.

### 4.4 Cuenta bloqueada

El bloqueo tiene prioridad sobre cualquier otra comprobación: si hay un bloqueo
vigente se responde `423` aunque la contraseña sea correcta y el email esté
verificado.

- El contador se reinicia a `0` con un login exitoso o al expirar la ventana.
- Los intentos fallidos son **consecutivos**: no se acumulan entre ventanas
  distintas.
- Al expirar el bloqueo la cuenta vuelve a estar disponible sola, sin
  intervención humana.

### 4.5 Intentos repetidos

El contador es **por cuenta**, no por IP. Esto tiene un coste conocido y
aceptado: un tercero que conozca un email puede bloquear esa cuenta 15 minutos a
propósito. Se asume en esta funcionalidad; la mitigación (rate limiting por IP,
CAPTCHA) queda fuera de alcance.

Los intentos contra un email **inexistente** no crean ni incrementan ningún
contador. Si lo hicieran, medir el bloqueo permitiría sondear qué emails están
registrados.

### 4.6 Cuenta desactivada

Una cuenta con `status = DISABLED` responde `403 ACCOUNT_DISABLED`, y solo
después de validar la contraseña — igual que con el email sin verificar.

La distinción entre desactivada y bloqueada importa, y por eso son campos
distintos: el bloqueo vive en `lockedUntil` y **caduca solo**; la desactivación
vive en `status` y solo la levanta un administrador. Una cuenta puede estar en
los dos estados a la vez.

### 4.7 Orden de las comprobaciones

El orden es deliberado y forma parte del contrato:

1. Validación del formato del cuerpo → `400`
2. Bloqueo vigente → `423`
3. Existencia de la cuenta y contraseña → `401`
4. Email verificado → `403`
5. Estado de la cuenta → `403`

Los pasos 4 y 5 van **después** de comprobar la contraseña, a propósito: quien
no conoce la credencial no puede averiguar si una cuenta existe, si está
verificada ni si está desactivada.

### 4.8 Token expirado

- **Access token caducado** → `401 TOKEN_EXPIRED`. La SPA reintenta **una sola
  vez** contra `/auth/refresh` y, si funciona, repite la petición original.
- **Refresh token caducado** → `401 TOKEN_EXPIRED`. La SPA descarta la sesión y
  lleva al formulario de login.
- **Cookie de refresh ausente, malformada o revocada** → `401 TOKEN_INVALID`.
- **Refresh token reutilizado después de haber sido rotado** → se trata como
  robo de credencial: se revoca **toda la familia** de refresh tokens de ese
  usuario y se responde `401 TOKEN_INVALID`. Todas sus sesiones se cierran.
- **Refrescos concurrentes** desde varias pestañas: la rotación es atómica. Una
  gana y las demás reciben `401`; el cliente reintenta una vez antes de
  descartar la sesión.

### 4.9 Email sin verificar

Se comprueba después de validar la contraseña (ver 4.7). El `code`
`EMAIL_NOT_VERIFIED` es distinguible a propósito, para que el frontend pueda
ofrecer "reenviar el correo de verificación" — aunque **ese reenvío no se
implementa aquí** (bloque 5).

### 4.10 El token que emite este login sale del módulo

El access token del escenario 1 no lo lee solo nuestra SPA: viaja dentro de las
peticiones que el usuario hace a los otros seis módulos del marketplace, que lo
verifican por su cuenta con la clave pública del JWKS.

Dos consecuencias para esta funcionalidad:

- **Se firma con RS256**, no con HS256, y sus claims —incluidos `permissions`,
  `type`, `iss` y `aud`— son contrato. Están fijados en
  [`api-contract.md` §1.4](api-contract.md) y **no pueden cambiarse
  unilateralmente**: hacerlo rompería a seis equipos a la vez.
- **Cerrar sesión no invalida el access token en los otros módulos.** El logout
  del escenario 8 revoca el refresh token, pero el access token sigue siendo
  criptográficamente válido hasta su `exp`. Es el desfase de 15 minutos que
  documenta [`integracion.md` §5.1](integracion.md), y quien no lo tolere usa la
  introspección.

## 5. Fuera de alcance

Nada de lo siguiente se implementa en esta funcionalidad. Si hace falta, se
especifica aparte.

- **Registro / alta de cuentas nuevas.** Esta spec asume usuarios que ya
  existen en la base de datos. Especificado aparte en
  [`registro.md`](registro.md).
- **Verificación de email:** el envío y reenvío del correo, y el endpoint que
  consume el token de verificación. Aquí solo se **lee** el flag
  `emailVerified`. Especificado aparte en [`registro.md`](registro.md).
- **Recuperación de contraseña** ("olvidé mi contraseña") y **cambio de
  contraseña** estando autenticado.
- **Segundo factor (MFA):** TOTP, SMS y códigos de respaldo.
- **Login federado / OAuth:** Google, Apple, GitHub.
- **"Recuérdame"** y sesiones de duración extendida más allá de los 30 días del
  refresh token.
- **Autorización.** Esta spec **devuelve** los roles del usuario, pero no define
  qué puede hacer cada rol ni protege ningún endpoint de negocio.
- **Desbloqueo manual de cuentas por un administrador**, y el panel de
  administración completo.
- **Rate limiting por IP, CAPTCHA y protección anti-bot** (ver el riesgo
  aceptado en 4.5).
- **Auditoría de accesos**, histórico de sesiones y aviso por correo de "nuevo
  inicio de sesión".
- **Revocación de access tokens ya emitidos.** Solo caducan, con una ventana
  máxima de 15 minutos. Únicamente el refresh token es revocable.
- **Límite de sesiones concurrentes** por usuario: se admiten todas las que
  quiera.
- **Internacionalización** de los mensajes de error: solo español.
