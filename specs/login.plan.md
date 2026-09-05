# Plan de implementación — Login con email y contraseña

Resumen de **qué se implementaría y con qué stack** para cumplir
[login.md](login.md) y [api-contract.md](api-contract.md). Cada decisión está
cerrada: no hay nada por definir.

Este documento describe la implementación; **no la ejecuta**. No se ha escrito
código todavía.

## 1. Stack

### Frontend — `./frontend`

| Pieza | Elección | Por qué |
| --- | --- | --- |
| Base | React 19 + TypeScript + Vite + Yarn | Fijado en el stack del proyecto |
| Rutas | `react-router-dom` v7 | Rutas protegidas y redirección tras el login |
| Formulario | `react-hook-form` + `zod` | El esquema `zod` valida en el navegador **y** se reutiliza en los tests, sin duplicar reglas |
| HTTP | Cliente `fetch` propio (~80 líneas) | Un interceptor de `401` es todo lo que hace falta; añadir Axios no aporta aquí |
| Estado de sesión | `AuthContext` con `useReducer` | Alcance de una funcionalidad: no justifica Redux ni Zustand |

**El access token se guarda solo en memoria (React state), nunca en
`localStorage` ni en `sessionStorage`.** Esa es la razón de ser de la cookie
`httpOnly`: si el token se guardase en almacenamiento accesible desde
JavaScript, un XSS lo robaría y todo el diseño perdería el sentido. Al recargar
la página se rehidrata con `GET /auth/me`, y si el access token ya caducó, con
`POST /auth/refresh` primero.

### Backend — `./backend`

Spring Boot 4.1 + WebFlux, Java 21, Maven. Generado desde `start.spring.io`.

| Dependencia | Para qué |
| --- | --- |
| `spring-boot-starter-webflux` | API reactiva |
| `spring-boot-starter-security` | `BCryptPasswordEncoder` y la cadena de filtros reactiva |
| `spring-boot-starter-data-r2dbc` + `r2dbc-postgresql` | Acceso a datos no bloqueante, coherente con WebFlux |
| `spring-boot-starter-validation` | Validación declarativa de los DTO de entrada |
| `lombok` | Menos repetición en entidades y DTO |
| `com.nimbusds:nimbus-jose-jwt` | Firma y verificación de los JWT (**RS256**) y publicación del JWKS |
| `flyway-core` + `postgresql` (JDBC) | Migraciones versionadas |

**Sobre Flyway y JDBC:** Flyway no habla R2DBC, así que el driver JDBC entra
solo para ejecutar las migraciones al arrancar. En tiempo de ejecución la
aplicación usa **únicamente** R2DBC; ninguna petición HTTP toca JDBC. Es el
precio de tener migraciones versionadas en un equipo de 6 personas, y se paga a
conciencia.

El par de claves RSA se lee de las variables de entorno `JWT_PRIVATE_KEY`,
`JWT_PUBLIC_KEY` y `JWT_KEY_ID`. No hay valor por defecto: si falta la clave
privada, la aplicación no arranca.

**RS256 y no HS256**, porque el token que emite este login lo verifican los otros
seis módulos del marketplace con la clave pública del JWKS. Con HS256 habría que
repartirles la clave de firma, y con ella podrían emitir tokens de administrador.
Ver [`stack.md` §4](stack.md) e [`integracion.md`](integracion.md).

## 2. Modelo de datos

```sql
-- V1__create_users.sql
CREATE TABLE users (
    id                    UUID PRIMARY KEY,
    email                 VARCHAR(254) NOT NULL UNIQUE,  -- siempre en minúsculas
    password_hash         VARCHAR(60)  NOT NULL,         -- BCrypt, coste 12
    display_name          VARCHAR(100) NOT NULL,
    email_verified        BOOLEAN      NOT NULL DEFAULT FALSE,
    status                VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE | DISABLED
    failed_login_attempts SMALLINT     NOT NULL DEFAULT 0,
    locked_until          TIMESTAMPTZ,
    created_at            TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role    VARCHAR(30) NOT NULL,
    PRIMARY KEY (user_id, role)
);

-- V2__create_refresh_tokens.sql
CREATE TABLE refresh_tokens (
    id         UUID PRIMARY KEY,
    user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash CHAR(64)    NOT NULL UNIQUE,  -- SHA-256 del token opaco
    family_id  UUID        NOT NULL,         -- detecta reutilización tras rotar
    expires_at TIMESTAMPTZ NOT NULL,
    revoked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_refresh_tokens_family ON refresh_tokens (family_id);
```

Detalles que importan:

- El refresh token es un valor aleatorio de 256 bits, y en la base de datos se
  guarda **solo su SHA-256**. Una filtración de la tabla no da sesiones
  utilizables. No lleva BCrypt porque no es una contraseña de baja entropía:
  con 256 bits aleatorios, el hash rápido basta.
- `family_id` implementa la detección de reutilización del caso límite 4.8: al
  rotar, el token nuevo hereda la familia del anterior. Si llega un token ya
  revocado, se revoca la familia entera.
- El bloqueo vive en `users` (`failed_login_attempts` + `locked_until`), no en
  memoria: sobrevive a un reinicio y funciona con varias instancias.

## 3. Estructura del backend

Paquete raíz `com.marketplace.accessmanager`, organizado **por funcionalidad**
y con tres capas dentro de cada una:

```
auth/
├── api/
│   ├── AuthController.java          # los 4 endpoints
│   └── dto/                         # LoginRequest, LoginResponse, UserResponse
├── application/
│   ├── LoginUseCase.java            # orquesta el orden de comprobaciones de 4.7
│   ├── RefreshTokenUseCase.java     # rotación + detección de reutilización
│   ├── LogoutUseCase.java
│   └── LoginAttemptPolicy.java      # 5 intentos / 15 min
├── domain/
│   ├── User.java, Role.java, AccountStatus.java, RefreshToken.java
│   ├── UserRepository.java          # puerto
│   ├── RefreshTokenRepository.java  # puerto
│   └── exception/                   # una excepción por code del catálogo
└── infrastructure/
    ├── persistence/                 # adaptadores R2DBC
    └── security/
        ├── JwtTokenService.java     # emisión y validación
        ├── RefreshTokenService.java # generación, hashing, rotación
        └── SecurityConfig.java      # SecurityWebFilterChain + BCrypt

shared/
└── error/GlobalExceptionHandler.java  # excepción -> ProblemDetail (RFC 7807)
```

`AuthController` usa controladores anotados, no `RouterFunction`: son cuatro
endpoints y la validación declarativa con `@Valid` sale gratis.

Todo devuelve `Mono<T>`. Ninguna llamada bloqueante en el hilo de eventos —
incluido BCrypt, que **sí** bloquea (~100 ms con coste 12) y por eso se envuelve
en `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`. Es el
error clásico al mezclar WebFlux con hashing de contraseñas.

## 4. Frontend: piezas concretas

```
src/
├── features/auth/
│   ├── api/authApi.ts               # login, refresh, logout, me
│   ├── components/LoginForm.tsx     # react-hook-form + zod
│   ├── context/AuthContext.tsx      # token en memoria + user
│   ├── hooks/useAuth.ts
│   ├── pages/LoginPage.tsx
│   └── schemas/loginSchema.ts       # compartido con los tests
├── shared/
│   ├── api/httpClient.ts            # fetch + interceptor 401 -> refresh
│   └── routing/ProtectedRoute.tsx
```

El interceptor reintenta **una sola vez** ante un `401 TOKEN_EXPIRED`, y encola
las peticiones concurrentes tras un único refresh en vuelo para no disparar
cuatro rotaciones a la vez. Todas las peticiones a `/api/v1/auth` van con
`credentials: "include"` para que viaje la cookie.

Cada `code` del catálogo se traduce a un mensaje en español en un único mapa; la
interfaz nunca muestra `detail` crudo. `ACCOUNT_LOCKED` muestra los minutos que
faltan a partir de `retryAfterSeconds`, y `EMAIL_NOT_VERIFIED` muestra un aviso
**sin** botón de reenvío, porque ese flujo está fuera de alcance.

## 5. Tests — uno por criterio de aceptación

### Backend

| Test | Cubre |
| --- | --- |
| `AuthControllerIT` con `WebTestClient` | Los 9 escenarios del bloque 3, de extremo a extremo |
| `LoginAttemptPolicyTest` | Contador, umbral de 5, ventana de 15 min, reinicio (4.4, 4.5) |
| `RefreshTokenServiceTest` | Rotación, reutilización, revocación de familia (4.8) |
| `EmailNormalizationTest` | Mayúsculas y espacios (4.2) |
| `PasswordLimitsTest` | Vacía y de más de 72 bytes, contando bytes y no caracteres (4.3) |
| `CheckOrderTest` | El orden exacto de comprobaciones de 4.7 |

Los tests de integración usan **Testcontainers con PostgreSQL real**, no H2: R2DBC
y los tipos de Postgres (`TIMESTAMPTZ`, `UUID`) se comportan distinto y un H2 en
memoria daría verde sobre una mentira.

La respuesta de tiempo constante (4.1) se verifica comparando la mediana de 50
peticiones con email existente contra 50 con email inexistente, con una
tolerancia amplia. Es un test frágil por naturaleza, así que se marca como
`@Tag("timing")` y se ejecuta fuera del pipeline de cada commit.

### Frontend

Vitest + React Testing Library + MSW simulando cada respuesta del catálogo:
`INVALID_CREDENTIALS`, `EMAIL_NOT_VERIFIED`, `ACCOUNT_DISABLED`,
`ACCOUNT_LOCKED`, `VALIDATION_ERROR`, `TOKEN_EXPIRED`. Se prueban además la
validación del formulario antes de enviar, el reintento único del interceptor, y
que la ruta protegida redirija al login sin sesión.

## 6. Orden de implementación

1. Migraciones Flyway + entidad `User` + repositorio R2DBC
2. `BCryptPasswordEncoder` sobre `boundedElastic` + `LoginUseCase` (escenarios 1-3)
3. `JwtTokenService` + `POST /auth/login` + `GlobalExceptionHandler`
4. `emailVerified` y `status` en el flujo, con el orden de 4.7 (escenarios 4, 6)
5. `LoginAttemptPolicy`: contador y bloqueo (escenarios 5, 6)
6. Refresh tokens con rotación y detección de reutilización (escenario 7)
7. `POST /auth/logout` y `GET /auth/me` (escenarios 8, 9)
8. Frontend: `LoginForm` + `AuthContext` + `authApi`
9. Frontend: interceptor de refresh y `ProtectedRoute`

Los pasos 1-7 son backend y los 8-9 frontend, así que a partir del paso 3 —con
el contrato ya congelado en `api-contract.md`— las dos mitades pueden avanzar en
paralelo contra MSW.

## 7. Riesgos asumidos

- **Bloqueo por cuenta:** un tercero puede bloquear una cuenta ajena 15 minutos
  (caso límite 4.5). Aceptado; la mitigación está fuera de alcance.
- **Access token no revocable:** una sesión comprometida sigue siendo válida
  hasta 15 minutos. Es el precio del modelo stateless.
- **Cookie `SameSite=Strict`:** si el frontend acaba en un dominio distinto al
  de la API, la cookie no viajará y habrá que pasar a `SameSite=None; Secure`
  con CORS con credenciales. Mientras compartan dominio, `Strict` es lo correcto.
