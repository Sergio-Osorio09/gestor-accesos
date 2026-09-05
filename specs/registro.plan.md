# Plan de implementación — Registro de cuentas y verificación de email

Resumen de **qué se implementaría y con qué stack** para cumplir
[registro.md](registro.md) y [api-contract.md](api-contract.md). Cada decisión
está cerrada: no hay nada por definir.

Este documento describe la implementación; **no la ejecuta**. De los diez pasos
del apartado 6, los **pasos 8 y 9 (el frontend) ya están implementados** en
`frontend/`; el backend, pasos 1 a 7, sigue sin escribirse.

Se apoya en lo que ya define [login.plan.md](login.plan.md) — tabla `users`,
BCrypt, migraciones Flyway, organización por funcionalidad — y solo detalla lo
que añade el registro.

## 1. Stack

Sin tecnologías nuevas respecto a [stack.md](stack.md), salvo el envío de
correo:

| Pieza | Elección | Por qué |
| --- | --- | --- |
| Envío de correo | `spring-boot-starter-mail` sobre SMTP | Estándar, sin atarnos a la API de ningún proveedor. Cambiar de proveedor es cambiar configuración, no código |
| SMTP en desarrollo | Mailpit en Docker | Captura todos los correos y los muestra en una web local; nadie recibe correos de prueba por accidente |
| Lista de contraseñas comunes | Fichero en `resources`, top 10 000 | Comprobación **local**: la contraseña no sale nunca de nuestro proceso |

> **JavaMail bloquea.** `JavaMailSender` es una API bloqueante, así que cada
> envío va envuelto en `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`,
> igual que BCrypt. Es la misma trampa de WebFlux que documenta `stack.md`.

El fichero de contraseñas comunes se carga una vez al arrancar en un
`Set<String>` inmutable (unos 100 KB en memoria). Comprobar pertenencia es O(1);
consultar un servicio externo por cada alta sería más lento y filtraría
contraseñas a un tercero.

## 2. Modelo de datos

Dos tablas nuevas. La tabla `users` no cambia: `email_verified` ya existe desde
`V1__create_users.sql`.

```sql
-- V3__create_email_verification_tokens.sql
CREATE TABLE email_verification_tokens (
    id           UUID PRIMARY KEY,
    user_id      UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash   CHAR(64)    NOT NULL UNIQUE,  -- SHA-256 del token opaco
    expires_at   TIMESTAMPTZ NOT NULL,
    consumed_at  TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_verification_tokens_user ON email_verification_tokens (user_id);

-- V4__create_email_outbox.sql
CREATE TABLE email_outbox (
    id              UUID PRIMARY KEY,
    recipient       VARCHAR(254) NOT NULL,
    template        VARCHAR(50)  NOT NULL,  -- VERIFICATION | ALREADY_REGISTERED
    payload         JSONB        NOT NULL,  -- variables de la plantilla
    attempts        SMALLINT     NOT NULL DEFAULT 0,
    next_attempt_at TIMESTAMPTZ  NOT NULL DEFAULT now(),
    sent_at         TIMESTAMPTZ,
    last_error      TEXT,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbox_pending ON email_outbox (next_attempt_at)
    WHERE sent_at IS NULL;
```

Detalles que importan:

- **El token se guarda solo como SHA-256**, igual que el refresh token. Una
  filtración de la tabla no permite verificar cuentas ajenas. No lleva BCrypt
  porque no es una contraseña de baja entropía: con 256 bits aleatorios, el
  hash rápido basta.
- **`consumed_at` en vez de borrar la fila.** Permite distinguir "token que
  nunca existió" de "token ya usado" en los logs, aunque la API responda lo
  mismo a los dos (caso límite 4.6 de la spec).
- **El cooldown de reenvío se deriva** de `MAX(created_at)` de los tokens de esa
  cuenta. No hace falta una columna extra en `users`.
- **Tabla outbox, y no un envío directo.** Es lo que hace cumplible la garantía
  del caso 4.8: el alta y el encolado ocurren **en la misma transacción**, así
  que si el correo falla la cuenta ya está creada y el envío se reintenta. Sin
  outbox, una caída del proveedor perdería altas o dejaría cuentas a medias.
- El índice parcial sobre `next_attempt_at` mantiene barata la consulta del
  poller aunque la tabla crezca: solo indexa lo que está sin enviar.

## 3. Estructura del backend

Todo cuelga del paquete `auth` que ya define `login.plan.md`, más un paquete
`notification` para el correo, que no es asunto de autenticación:

```
auth/
├── api/
│   ├── RegistrationController.java   # los 3 endpoints nuevos
│   └── dto/                          # RegisterRequest, VerifyEmailRequest,
│                                     # ResendVerificationRequest, AcceptedResponse
├── application/
│   ├── RegisterUserUseCase.java      # alta + token + encolado, en una transacción
│   ├── VerifyEmailUseCase.java       # consume el token y marca el email
│   ├── ResendVerificationUseCase.java# cooldown + rotación del token
│   └── PasswordPolicy.java           # longitud, lista de comunes, similitud
├── domain/
│   ├── EmailVerificationToken.java
│   └── EmailVerificationTokenRepository.java   # puerto
└── infrastructure/
    ├── persistence/                  # adaptador R2DBC del puerto
    └── security/
        └── VerificationTokenService.java   # genera, hashea, valida, consume

notification/
├── domain/EmailMessage.java, EmailTemplate.java
├── application/EmailOutboxService.java     # encola
└── infrastructure/
    ├── OutboxDispatcher.java               # poller con reintentos
    └── SmtpEmailSender.java                # JavaMailSender en boundedElastic
```

**`RegistrationController` aparte de `AuthController`.** Siete endpoints en una
sola clase empiezan a ser difíciles de leer, y con 6 personas en el equipo dos
clases separadas reducen los conflictos de merge.

**El despachador del outbox** es un `Flux.interval(Duration.ofSeconds(10))` que
toma las filas con `sent_at IS NULL` y `next_attempt_at <= now()`, las envía y
aplica retroceso exponencial (1 min, 5 min, 15 min, 1 h, 6 h) hasta 5 intentos.
Se elige `Flux.interval` sobre `@Scheduled` para no salir del modelo reactivo.

**Con varias instancias**, la toma de filas usa
`SELECT ... FOR UPDATE SKIP LOCKED`, así dos instancias no envían el mismo
correo dos veces.

**`RegisterUserUseCase` no consulta antes de insertar.** Intenta el `INSERT` y
captura la violación de la restricción `UNIQUE`; ese camino encola el correo de
"alguien intentó registrarse" y devuelve el mismo `202`. Es lo que cierra la
carrera del caso límite 4.3 de la spec.

## 4. Frontend: piezas concretas

```
src/features/auth/
├── api/authApi.ts               # + register, verifyEmail, resendVerification
├── components/
│   ├── RegisterForm.tsx         # react-hook-form + zod
│   └── PasswordField.tsx        # medidor de fuerza y requisitos visibles
├── pages/
│   ├── RegisterPage.tsx
│   ├── CheckYourEmailPage.tsx   # pantalla tras el 202, con botón de reenvío
│   └── VerifyEmailPage.tsx      # lee ?token= de la URL y lo envía
└── schemas/registerSchema.ts    # compartido con los tests
```

**`CheckYourEmailPage` es obligatoria, no decorativa.** Como el registro
responde `202` sin decir si la cuenta se creó, la interfaz **no puede** afirmar
"cuenta creada": muestra "si la dirección es válida, te hemos enviado un
correo". Es donde se paga la contrapartida de la anti-enumeración, y la
redacción del texto es parte del diseño.

**`VerifyEmailPage`** saca el token del query string, lo envía y cubre tres
estados: éxito (con enlace al login), token caducado (con botón de reenvío) y
token inválido. El token **nunca** se guarda en `localStorage`.

El medidor de fuerza es solo orientativo: **la validación que manda es la del
backend**. El navegador no tiene la lista de contraseñas comunes; duplicarla en
el bundle añadiría 100 KB de descarga a cada visita.

El cooldown de 60 s se refleja con un botón deshabilitado y una cuenta atrás,
pero el frontend **no** es la defensa: si llega un `429 RESEND_TOO_SOON`, se
muestra el `retryAfterSeconds` de la respuesta.

## 5. Tests — uno por criterio de aceptación

### Backend

| Test | Cubre |
| --- | --- |
| `RegistrationControllerIT` con `WebTestClient` | Los 12 escenarios del bloque 3, de extremo a extremo |
| `AntiEnumerationTest` | Que las respuestas de los escenarios 1 y 2 son **idénticas byte a byte**, y también las del reenvío sobre cuenta inexistente y ya verificada |
| `PasswordPolicyTest` | Mínimo de 12, lista de comunes, límite de 72 **bytes** contando bytes y no caracteres, similitud con email y nombre (4.4, 4.5) |
| `VerificationTokenServiceTest` | Un solo uso, caducidad de 24 h, rotación al reenviar, cuenta ya verificada (4.6) |
| `ConcurrentRegistrationTest` | Dos altas simultáneas con el mismo email: una crea, la otra cae en el camino del escenario 2 (4.3) |
| `ResendCooldownTest` | Los 60 s y el `retryAfterSeconds` del `429` (4.7) |
| `EmailOutboxIT` | La cuenta se crea aunque el envío falle, y el reintento con retroceso (4.8) |
| `DisplayNameNormalizationTest` | NFC, recorte de espacios, rechazo de caracteres de control (4.10) |

Los tests de integración usan **Testcontainers con PostgreSQL real** y
**GreenMail** como servidor SMTP en memoria, para verificar que el correo se
envía de verdad sin depender de un servicio externo.

El `ConcurrentRegistrationTest` lanza las dos peticiones con un `CountDownLatch`
compartido para que salgan a la vez. Es el único test que puede volverse
intermitente; si lo hace, se repite N veces en lugar de relajar la aserción.

### Frontend

Vitest + React Testing Library + MSW simulando `202`, `400 VALIDATION_ERROR`,
`400 WEAK_PASSWORD`, `410 VERIFICATION_TOKEN_EXPIRED`,
`410 VERIFICATION_TOKEN_INVALID` y `429 RESEND_TOO_SOON`. Se prueba además que
`CheckYourEmailPage` **no afirma** que la cuenta se haya creado — es una
aserción sobre el texto, y protege la decisión de producto de la spec.

## 6. Orden de implementación

1. Migraciones V3 y V4 + entidades y repositorios R2DBC
2. `PasswordPolicy` con la lista de contraseñas comunes en `resources`
3. `EmailOutboxService` y `SmtpEmailSender` con Mailpit en local
4. `RegisterUserUseCase` + `POST /auth/register` (escenarios 1-7)
5. `OutboxDispatcher` con reintentos y retroceso exponencial (4.8)
6. `VerificationTokenService` + `POST /auth/verify-email` (escenarios 8-10)
7. `POST /auth/resend-verification` con cooldown (escenario 11)
8. Frontend: `RegisterForm`, `RegisterPage`, `CheckYourEmailPage`
9. Frontend: `VerifyEmailPage` y el reenvío desde la pantalla de espera
10. Prueba conjunta con `login.md`: registrar, verificar y entrar (escenario 12)

Del paso 4 en adelante, con el contrato ya congelado, frontend y backend pueden
avanzar en paralelo contra MSW.

**Este trabajo va antes que el de `login.plan.md`**: sin registro no hay cuentas
que puedan iniciar sesión. Las migraciones V1 y V2 y la entidad `User`, que
`login.plan.md` lista como su paso 1, se adelantan aquí.

## 7. Riesgos asumidos

- **Peor UX a cambio de no filtrar cuentas.** Quien ya tiene cuenta y lo olvidó
  no lo ve en pantalla: se entera por correo. Es el precio de la decisión de la
  spec, y se paga entero en `CheckYourEmailPage`.
- **Alias del mismo buzón** permiten crear varias cuentas (`ada+1@`, `ada+2@`).
  Aceptado: canonizar alias por proveedor fusionaría cuentas de personas
  distintas, que es un fallo peor.
- **Cuentas sin verificar acumulándose.** Nada las borra: su limpieza está fuera
  de alcance en la spec. Con el tiempo la tabla `users` crece con cuentas
  muertas, y habrá que especificar una purga.
- **Sin CAPTCHA ni límite por IP**, un bot puede dar de alta cuentas en masa. El
  cooldown de reenvío es por cuenta, no por origen, así que no lo impide.
- **El outbox garantiza la entrega al SMTP, no al buzón.** Si el proveedor
  acepta el correo y luego rebota, nadie se entera: la gestión de rebotes está
  fuera de alcance.
