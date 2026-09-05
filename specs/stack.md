# Stack tecnológico

Decisiones de tecnología del proyecto y por qué se tomaron. Cambiar cualquiera
de estas piezas requiere actualizar este documento primero.

## 1. Frontend

| Pieza | Elección | Versión |
| --- | --- | --- |
| Librería de UI | React | 19 |
| Lenguaje | TypeScript | 5.x |
| Empaquetador | Vite | 7.x |
| Gestor de paquetes | Yarn | 4.x (vía corepack) |

**React** — es el estándar de facto para SPAs, y el equipo ya lo conoce. Un
gestor de accesos es sobre todo formularios y estado de sesión: nada que exija
un framework más pesado.

**TypeScript** — un módulo de seguridad es justo donde un `undefined`
inesperado se convierte en un fallo de autenticación. Los tipos del contrato de
API (`specs/api-contract.md`) se escriben una vez y el compilador verifica cada
uso.

**Vite** — arranque casi instantáneo y recarga en caliente, lo que importa con
6 personas iterando. Trae `create vite --template react-ts` ya configurado, así
que no hay que montar un Webpack a mano.

**Yarn** — elección del equipo. Se activa con `corepack enable`, que viene
incluido en Node 20+, así que **no hace falta instalar Yarn globalmente**. El
`yarn.lock` se commitea siempre.

## 2. Backend

| Pieza | Elección | Versión |
| --- | --- | --- |
| Lenguaje | Java | 21 (LTS) |
| Framework | Spring Boot | 4.1.x |
| Modelo web | Spring WebFlux | reactivo |
| Construcción | Maven (con wrapper `mvnw`) | 3.9.x |

**Java 21** — versión LTS con soporte a largo plazo. Trae `records` y
pattern matching, que quitan mucho ruido de los DTO.

**Spring Boot** — ecosistema maduro para autenticación: Spring Security aporta
BCrypt y la cadena de filtros ya resueltos y auditados. Reimplementar eso a
mano en un módulo de seguridad sería un error.

**WebFlux** — un gestor de accesos es un servicio de E/S: espera a la base de
datos y al proveedor de correo mucho más de lo que calcula. El modelo reactivo
atiende muchas conexiones concurrentes sin un hilo por petición, que es
exactamente el perfil de carga de un pico de logins.

> **La contrapartida, y hay que tenerla presente:** en WebFlux **ninguna
> llamada bloqueante puede ejecutarse en el hilo de eventos**. BCrypt bloquea
> (~100 ms), así que va envuelto en `Schedulers.boundedElastic()`. Es el error
> más común al mezclar Spring Security con WebFlux.

**Maven con wrapper** — el `mvnw` que genera Spring Initializr fija la versión
de Maven en el repositorio, así que **no hace falta instalar Maven**: todo el
equipo construye con la misma versión.

## 3. Persistencia

| Pieza | Elección |
| --- | --- |
| Base de datos | PostgreSQL 16 |
| Driver | R2DBC (`r2dbc-postgresql`) |
| Migraciones | Flyway |

**PostgreSQL** — los datos de un gestor de accesos son relacionales (usuarios,
roles, sesiones) y necesitan garantías fuertes: unicidad real del email y
transacciones al rotar tokens.

**R2DBC** — driver reactivo no bloqueante. Usar JPA/Hibernate aquí sería
contradecir la elección de WebFlux: es bloqueante y obligaría a aislarlo en un
scheduler aparte, perdiendo la ventaja del modelo reactivo.

**Flyway** — migraciones versionadas en el repositorio, imprescindible con 6
personas tocando el esquema.

> **Nota:** Flyway no habla R2DBC, así que el driver JDBC de PostgreSQL entra
> como dependencia **solo para ejecutar las migraciones al arrancar**. En
> tiempo de ejecución la aplicación usa únicamente R2DBC; ninguna petición HTTP
> toca JDBC.
>
> **Estas dependencias todavía no están en el `pom.xml`.** Se añaden al
> implementar el login, que es la primera funcionalidad que necesita
> persistencia. Así el esqueleto arranca sin exigir una base de datos.

## 4. Seguridad

| Pieza | Elección |
| --- | --- |
| Hash de contraseñas | BCrypt, coste 12 |
| Access token | JWT firmado con HS256, 15 min |
| Refresh token | Valor opaco de 256 bits en cookie `httpOnly`, 30 días |
| Librería JWT | `io.jsonwebtoken:jjwt` 0.12.x |

El detalle del modelo de sesión está en
[api-contract.md](api-contract.md#14-autenticación).

## 5. Pruebas

| Capa | Herramientas |
| --- | --- |
| Backend unitario | JUnit 5 + Mockito |
| Backend integración | `WebTestClient` + Testcontainers (PostgreSQL real) |
| Frontend | Vitest + React Testing Library + MSW |

**Testcontainers en vez de H2** — R2DBC y los tipos propios de PostgreSQL
(`TIMESTAMPTZ`, `UUID`) se comportan distinto en H2. Un test en memoria daría
verde sobre un comportamiento que no es el de producción.

## 6. Herramientas del entorno local

Lo único que hay que tener instalado:

| Herramienta | Versión mínima | Notas |
| --- | --- | --- |
| Node.js | 20 | Yarn se activa con `corepack enable` |
| JDK | 21 | Maven llega con el wrapper `./mvnw` |
| Git | 2.30 | — |
| Docker | opcional | Solo para levantar PostgreSQL al implementar el login |
