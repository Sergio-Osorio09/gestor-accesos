# Plan de implementación — API de identidad para los demás módulos

Resumen de **qué se implementaría y con qué stack** para cumplir
[integracion.md](integracion.md) y [api-contract.md](api-contract.md). Cada
decisión está cerrada: no hay nada por definir.

Este documento describe la implementación; **no la ejecuta**. Lo único ya hecho
es el contrato: [`openapi.yaml`](openapi.yaml) y el entorno simulado del
apartado 7.

Se apoya en lo que ya definen [login.plan.md](login.plan.md) y
[registro.plan.md](registro.plan.md) —tabla `users`, migraciones Flyway,
organización por funcionalidad— y solo detalla lo que añade la integración.

## 1. Stack

Una dependencia nueva y un cambio respecto a [stack.md](stack.md):

| Pieza | Elección | Por qué |
| --- | --- | --- |
| Firma y JWKS | `com.nimbusds:nimbus-jose-jwt` | Es lo que Spring Security ya usa por debajo. Publicar el JWKS y rotar claves sale de la caja en vez de escribirse a mano |
| Servidor de recursos | `spring-boot-starter-oauth2-resource-server` | Valida el `Bearer` y resuelve los scopes con anotaciones, sin filtro propio |
| Documentación | `springdoc-openapi-starter-webflux-ui` | Genera el OpenAPI **desde el código**, para que no se desincronice |

**Sustituye a `jjwt`**, que `login.plan.md` listaba para HS256. La firma pasa a
RS256 porque el token lo verifican los otros seis módulos con la clave pública;
el razonamiento completo está en [stack.md §4](stack.md).

El par de claves RSA de 2048 bits llega por `JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY` y
`JWT_KEY_ID`. **Sin clave privada la aplicación no arranca:** un módulo de
identidad que arranque sin poder firmar es peor que uno que no arranque.

## 2. Modelo de datos

Dos tablas nuevas. Ninguna toca las existentes.

```sql
-- V5__create_service_clients.sql
CREATE TABLE service_clients (
    id             UUID PRIMARY KEY,
    client_id      VARCHAR(64)  NOT NULL UNIQUE,   -- p. ej. dispatch-module
    secret_hash    VARCHAR(60)  NOT NULL,          -- BCrypt, coste 12
    module_name    VARCHAR(100) NOT NULL,
    scopes         TEXT[]       NOT NULL,
    enabled        BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at     TIMESTAMPTZ  NOT NULL DEFAULT now(),
    last_used_at   TIMESTAMPTZ
);

-- V6__create_integration_audit.sql
CREATE TABLE integration_audit (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    client_id      VARCHAR(64)  NOT NULL,
    endpoint       VARCHAR(200) NOT NULL,
    subject_id     UUID,                           -- usuario consultado, si aplica
    outcome        VARCHAR(30)  NOT NULL,          -- OK | INSUFFICIENT_SCOPE | NOT_FOUND
    sensitive_data BOOLEAN      NOT NULL DEFAULT FALSE,
    occurred_at    TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE INDEX idx_integration_audit_client ON integration_audit (client_id, occurred_at DESC);
```

Detalles que importan:

- **El `client_secret` se guarda con BCrypt**, igual que una contraseña de
  usuario. Es una credencial de baja entropía elegida por una persona, no un
  valor aleatorio de 256 bits: aquí el hash rápido no basta.
- **Los scopes viven en la fila del cliente, no en el token que él pide.** Así
  revocar un scope tiene efecto en cuanto caduque su token actual, sin tocar
  ninguna configuración.
- **`enabled` en vez de borrar la fila.** Revocar un módulo comprometido es un
  `UPDATE`, y la auditoría de lo que hizo sigue teniendo a quién referirse.
- **La auditoría es de solo inserción** y se indexa por cliente: la pregunta que
  se hará de verdad es "qué consultó el módulo de chatbot la semana pasada".
- **`sensitive_data`** distingue si el documento se entregó en claro o
  enmascarado, que es justo lo que el caso 5.9 de la spec obliga a registrar.

## 3. Estructura del backend

Un paquete nuevo, `integration`, hermano de `auth`. La integración **no es una
funcionalidad de autenticación**: es la frontera con otros sistemas, y mezclarla
con `auth` acabaría con un `AuthController` de quince endpoints.

```
integration/
├── api/
│   ├── IntrospectionController.java     # POST /auth/introspect
│   ├── ServiceTokenController.java      # POST /auth/token
│   ├── UserQueryController.java         # GET /users/{id}, POST /users/batch
│   ├── JwksController.java              # GET /.well-known/jwks.json
│   └── dto/                             # IntrospectionResponse, BatchRequest…
├── application/
│   ├── IntrospectTokenUseCase.java      # firma + estado ACTUAL del usuario
│   ├── IssueServiceTokenUseCase.java    # client_credentials
│   ├── QueryUsersUseCase.java           # individual y lote, con notFound
│   └── FieldMaskingPolicy.java          # mínima exposición por scope
├── domain/
│   ├── ServiceClient.java, Scope.java
│   └── ServiceClientRepository.java     # puerto
└── infrastructure/
    ├── persistence/                     # adaptador R2DBC
    ├── security/
    │   ├── RsaKeyProvider.java          # carga el par y publica el JWKS
    │   └── ScopeAuthorizationFilter.java
    └── audit/IntegrationAuditService.java
```

`auth/infrastructure/security/JwtTokenService.java`, que `login.plan.md` ya
prevé, **pasa a firmar con RS256** usando el `RsaKeyProvider` de aquí. Es el
único punto en el que los dos paquetes se tocan.

**El orden de comprobaciones del filtro de scopes es fijo**, y por la misma razón
que en `registro.md` §4.11: primero se valida el token de servicio, después el
scope, y solo al final se mira si el usuario consultado existe. Invertirlo
convertiría un `404` en una forma de enumerar cuentas sin credenciales.

**La introspección no consulta solo la firma.** Verifica el token *y* relee el
estado del usuario en la base de datos: es lo único que distingue esta vía de la
validación local, y sin esa lectura el endpoint no sirve para nada.

## 4. Aprovisionamiento de los clientes de servicio

Seis clientes, uno por módulo consumidor, con sus scopes mínimos:

| `client_id` | Scopes |
| --- | --- |
| `products-module` | *(ninguno: le basta la validación local)* |
| `sales-module` | `tokens:introspect`, `users:read`, `users:read:document` |
| `dispatch-module` | `users:read`, `addresses:read` |
| `chatbot-module` | `users:read` |
| `retail-module` | `users:read` |
| *(sexto, por confirmar)* | por acordar |

Se cargan con una migración de datos en los entornos de desarrollo, y en la nube
con un comando de administración: los secretos de producción **no van en una
migración versionada**.

Que productos aparezca sin scopes no es un olvido: si solo necesita saber quién
llama y qué permisos tiene, eso ya viaja en el token y no debe poder consultarnos
nada.

## 5. Tests — uno por criterio de aceptación

| Test | Cubre |
| --- | --- |
| `JwksIT` | El documento publica la clave, y durante una rotación publica las dos (escenario 13) |
| `LocalValidationTest` | Un token emitido por el login verifica contra la clave del JWKS, y uno firmado con otra clave no (escenario 1) |
| `ServiceTokenIT` | `client_credentials` devuelve solo los scopes concedidos aunque se pidan más; secreto incorrecto da `INVALID_CLIENT` (escenario 2) |
| `IntrospectionIT` | Token válido devuelve roles y permisos **vigentes**, no los del token (escenario 3) |
| `IntrospectionInactiveUserTest` | Usuario desactivado tras la emisión → `active: false`, `USER_INACTIVE`, con la firma todavía válida (escenario 4) |
| `UserQueryIT` | Datos básicos sin hash de contraseña (escenario 5) |
| `BatchQueryIT` | 10 identificadores con 2 inexistentes → `200`, 8 en `users` y 2 en `notFound`; duplicados colapsados; >100 → `BATCH_TOO_LARGE`; UUID malformado → `VALIDATION_ERROR` (escenario 6, casos 5.3) |
| `FieldMaskingTest` | Documento en claro con `users:read:document`, enmascarado a 3 dígitos sin él (escenarios 8 y 9) |
| `ScopeEnforcementTest` | `401` sin token y `403` con scope insuficiente, **antes** de mirar si el usuario existe (escenarios 10 y 11, caso 5.4) |
| `IntegrationAuditIT` | Todo acceso a dato sensible y todo `403` quedan registrados con módulo, endpoint y fecha (caso 5.9) |
| `ContractTest` | Las respuestas reales coinciden con [`openapi.yaml`](openapi.yaml) |

El `ScopeEnforcementTest` es el que protege la anti-enumeración desde otros
módulos: comprueba que la respuesta a un identificador inexistente **sin
credenciales** es byte a byte la misma que a uno existente.

Los tests de integración usan **Testcontainers con PostgreSQL real**, como el
resto del proyecto.

El `ContractTest` es el que evita el fallo clásico de este tipo de módulo:
que el OpenAPI publicado a seis equipos deje de describir lo que la API hace de
verdad.

## 6. Orden de implementación

1. `RsaKeyProvider` + `JwksController`, y cambiar la firma del login a RS256
2. Migración V5 + `ServiceClient`, repositorio R2DBC y aprovisionamiento
3. `POST /auth/token` con `client_credentials`
4. `ScopeAuthorizationFilter` y el orden de comprobaciones del apartado 3
5. `POST /auth/introspect`, con la relectura del estado del usuario
6. `GET /users/{id}` y `POST /users/batch` con `notFound`
7. Migración V6 + auditoría de acceso entre módulos
8. `FieldMaskingPolicy` y el enmascarado del documento
9. springdoc, y comparar el OpenAPI generado con el escrito a mano

**El paso 1 va antes que todo lo demás**, incluido el login: cambiar la firma
después de emitir tokens HS256 obligaría a invalidar las sesiones vivas. Como
todavía no hay ninguna, ahora sale gratis.

Los pasos 6 y 8 dependen de la spec de atributos de usuario, todavía sin
escribir: hasta que exista, `/users/{id}` devuelve solo los campos que ya tiene
la tabla `users`.

## 7. El entorno simulado, que ya existe

Es lo que desbloquea a los otros equipos sin una sola línea de backend:

```bash
npx @stoplight/prism-cli mock specs/openapi.yaml -p 4010
```

Sirve los ejemplos reales del contrato, incluidos los caminos de error, y admite
la cabecera `Prefer` para forzar un caso concreto. Está probado de extremo a
extremo: token de servicio, JWKS, consulta individual, lote parcial, `401`,
`403` y `BATCH_TOO_LARGE`.

Cuando la implementación esté lista, los consumidores solo cambian la URL base.
Las instrucciones para ellos están en [kit-integracion.md](kit-integracion.md).

## 8. Riesgos asumidos

- **La ventana de 15 minutos.** Un token sigue valiendo aunque el usuario haya
  sido desactivado. Es el precio de RS256 y validación local, está documentado en
  [integracion.md §5.1](integracion.md), y se cierra de verdad cuando existan los
  eventos asíncronos. Hasta entonces, quien no la tolere paga una llamada de red.
- **Sin límite de tasa por consumidor.** Un módulo mal programado que llame a la
  introspección en cada petición puede saturarnos. Está fuera de alcance, y la
  mitigación real es que el kit explique por qué no debe hacerlo.
- **Los códigos de rol sin acordar.** Viajan dentro del token y el plan del curso
  los define en español. Si el acuerdo llega tarde, cambiarlos después obliga a
  coordinar con seis equipos a la vez.
- **`permissions` vacío** hasta que exista el catálogo de permisos. Los
  consumidores autorizan por `roles` mientras tanto, y tendrán que revisar su
  código cuando el catálogo aparezca.
- **El OpenAPI escrito a mano puede desviarse** de la implementación entre el
  paso 1 y el paso 9. El `ContractTest` es lo que acota esa ventana.
