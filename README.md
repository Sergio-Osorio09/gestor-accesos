# gestor-accesos

Módulo de seguridad y autenticación de usuarios de un marketplace: registro,
inicio de sesión, gestión de sesiones y control de permisos.

Desarrollado con **SDD (Spec-Driven Development)**: la especificación se
escribe y se aprueba antes que el código. Si el código se desvía de la spec, se
corrige el código o se actualiza la spec de forma explícita — nunca a
posteriori y en silencio.

## Arquitectura

Dos aplicaciones desplegadas por separado, comunicadas solo por HTTP siguiendo
[`specs/api-contract.md`](specs/api-contract.md). No comparten código ni base de
datos.

```mermaid
graph TB
    subgraph navegador["🌐 Navegador"]
        SPA["React SPA<br/>React 19 · TypeScript · Vite<br/>Access token en memoria"]
        COOKIE[("Cookie httpOnly<br/>refresh_token<br/>JavaScript no puede leerla")]
    end

    subgraph servidor["☕ API — Spring Boot 4.1 + WebFlux · puerto 8080"]
        SEC["SecurityWebFilterChain<br/>Valida la cabecera Bearer"]
        CTRL["AuthController<br/>/api/v1/auth/*"]
        UC["Casos de uso<br/>Login · Refresh · Logout"]
        JWT["JwtTokenService<br/>Firma RS256 · 15 min"]
        HASH["PasswordHasher<br/>BCrypt en boundedElastic"]
        REPO["Repositorios R2DBC<br/>No bloqueantes"]
    end

    DB[("🐘 PostgreSQL 16<br/>users · user_roles<br/>refresh_tokens")]

    SPA -->|"HTTP + JSON<br/>Authorization: Bearer"| SEC
    COOKIE -.->|"El navegador la adjunta sola,<br/>solo en /api/v1/auth"| SEC
    SEC --> CTRL
    CTRL --> UC
    UC --> JWT
    UC --> HASH
    UC --> REPO
    REPO -->|"R2DBC reactivo"| DB
    SPA -.->|"El navegador la guarda;<br/>el código nunca la toca"| COOKIE

    classDef front fill:#61dafb,stroke:#1a7f9c,color:#000
    classDef back fill:#6db33f,stroke:#3d6b21,color:#fff
    classDef data fill:#336791,stroke:#1a3d5c,color:#fff
    class SPA front
    class SEC,CTRL,UC,JWT,HASH,REPO back
    class COOKIE,DB data
```

### Por qué está montado así

**Dos despliegues, no uno.** La SPA es estática y se sirve desde un CDN o un
nginx; la API es un proceso Java. Escalan y se despliegan por separado. En
desarrollo, el proxy de Vite redirige `/api` al backend, así que el navegador ve
un único origen y no hace falta configurar CORS.

**Dos credenciales con responsabilidades distintas.** El access token es un JWT
de 15 minutos que vive **solo en memoria** de la SPA; el refresh token es un
valor opaco de 30 días en una cookie `httpOnly` que JavaScript no puede leer.
Si el access token se guardase en `localStorage`, un XSS lo robaría y todo el
diseño perdería el sentido.

**El backend no guarda estado de sesión en memoria.** El bloqueo de cuentas y
los refresh tokens viven en PostgreSQL, así que se pueden levantar varias
instancias detrás de un balanceador sin sesiones pegajosas, y un reinicio no
borra un bloqueo en curso.

**Nada bloqueante en el hilo de eventos.** Es la regla que impone WebFlux.
BCrypt bloquea unos 100 ms, así que va envuelto en `Schedulers.boundedElastic()`
— el error más común al mezclar Spring Security con WebFlux.

> **Estado real:** hoy solo existen la SPA y la API, comunicadas por
> `/api/v1/status`. Los componentes de autenticación y PostgreSQL están
> especificados pero **sin implementar**; llegan con
> [`specs/login.md`](specs/login.md).

Los diagramas de secuencia del login y de la renovación de sesión están en
[`specs/arquitectura.md`](specs/arquitectura.md).

## Estructura: tres repositorios independientes

Este repositorio contiene **solo la documentación**. El frontend y el backend
viven en repositorios Git separados, con su propio historial:

| Repositorio | Contenido |
| --- | --- |
| [`gestor-accesos`](https://github.com/Sergio-Osorio09/gestor-accesos) | Specs y documentación (este repo) |
| [`gestor-accesos-frontend`](https://github.com/Sergio-Osorio09/gestor-accesos-frontend) | SPA en React + TypeScript + Vite |
| [`gestor-accesos-backend`](https://github.com/Sergio-Osorio09/gestor-accesos-backend) | API en Spring Boot + WebFlux |

**No se usan submódulos.** El repo de specs ignora `frontend/` y `backend/` a
propósito, para que cada equipo trabaje sin pisarse.

## Montar la estructura en local

Clona los tres repos de forma que el frontend y el backend queden **dentro** de
la carpeta de specs:

```bash
git clone https://github.com/Sergio-Osorio09/gestor-accesos.git
cd gestor-accesos

git clone https://github.com/Sergio-Osorio09/gestor-accesos-frontend.git frontend
git clone https://github.com/Sergio-Osorio09/gestor-accesos-backend.git backend
```

El resultado:

```
gestor-accesos/
├── .gitignore          # ignora frontend/ y backend/
├── README.md
├── AGENTS.md           # guía para agentes de IA y personas
├── CLAUDE.md
├── specs/
│   ├── overview.md
│   ├── stack.md
│   ├── api-contract.md
│   ├── arquitectura.md
│   ├── login.md
│   └── login.plan.md
├── frontend/           # repo Git independiente
└── backend/            # repo Git independiente
```

Comprueba que la separación funciona — `git status` en la raíz **no** debe
mencionar `frontend/` ni `backend/`:

```bash
git status
```

> **Importante:** cualquier commit sobre el frontend o el backend se hace
> entrando en su carpeta (`cd frontend && git commit ...`), **nunca desde la
> raíz**.

## Requisitos

| Herramienta | Versión | Comprobar |
| --- | --- | --- |
| Node.js | ≥ 20 | `node --version` |
| JDK | 21 | `javac -version` |
| Git | ≥ 2.30 | `git --version` |

**Yarn no se instala aparte**: se activa con corepack, incluido en Node 20+.

```bash
corepack enable
```

Si `javac` no aparece, tienes solo el JRE. En Debian/Ubuntu:

```bash
sudo apt install openjdk-21-jdk
```

## Arrancar en local

Hacen falta **dos terminales**.

### Backend — puerto 8080

```bash
cd backend
./mvnw spring-boot:run
```

No necesita base de datos: el esqueleto todavía no tiene persistencia. Se
añadirá PostgreSQL al implementar el login.

Comprueba que responde:

```bash
curl http://localhost:8080/api/v1/status
# {"service":"access-manager","status":"UP","apiVersion":"v1","timestamp":"..."}
```

### Frontend — puerto 5173

```bash
cd frontend
yarn install
yarn dev
```

Abre <http://localhost:5173>. La página muestra el estado de la conexión con la
API: en verde si el backend está levantado, en rojo si no. Desde ahí se llega a
`/register`, el formulario de alta.

Los tests del frontend no necesitan el backend —simulan la API con MSW—:

```bash
cd frontend
yarn test
```

El servidor de Vite hace de proxy de `/api` hacia `http://localhost:8080`, así
que el navegador ve un único origen y no hay que configurar CORS en
desarrollo.

## Estado actual

- ✅ Specs escritas: visión, stack, contrato de API, arquitectura y dos
  funcionalidades (registro y login).
- ✅ Frontend y backend arrancan y se comunican vía `/api/v1/status`.
- 🔶 Registro y verificación de email
  ([`specs/registro.md`](specs/registro.md)): **el frontend está implementado**
  —rutas `/register`, `/check-your-email` y `/verify-email`, con sus tests
  contra MSW—; **el backend no**, así que el flujo todavía no se puede
  completar de verdad.
- ⬜ Login: especificado en [`specs/login.md`](specs/login.md) y planificado en
  [`specs/login.plan.md`](specs/login.plan.md), **sin implementar**.
- 🔶 API de identidad para los demás módulos
  ([`specs/integracion.md`](specs/integracion.md)): **el contrato está publicado**
  en [`specs/openapi.yaml`](specs/openapi.yaml) y hay un entorno simulado
  funcionando; la implementación no ha empezado.

El registro va primero: `login.md` asume cuentas que ya existen, así que sin
alta no hay nadie que pueda iniciar sesión.

Lo siguiente son los pasos 1-7 de
[`specs/registro.plan.md`](specs/registro.plan.md): migraciones, política de
contraseñas, outbox de correo y los tres endpoints. Hacen falta **JDK 21** (no
solo el JRE) y **Docker**, para Testcontainers y Mailpit.

## Integración con los demás módulos del marketplace

Este módulo es el proveedor de identidad de los otros seis, y **ninguno puede
acceder a su base de datos**: toda relación pasa por API. Los otros equipos se
relacionan con nosotros de tres formas:

| Vía | Cuándo | Coste |
| --- | --- | --- |
| **Validación local del token** con la clave pública del JWKS | En cada petición ordinaria. Es el caso normal | Ninguna llamada de red. No detecta cambios de estado hasta que el token vence |
| **Llamada síncrona** a introspección o consulta de usuarios | Antes de anular, reembolsar o cambiar precios | Una llamada de red y una dependencia de nuestra disponibilidad |
| **Eventos asíncronos** | Para enterarse de bajas y bloqueos sin preguntar | **Todavía no existen**: se especifican aparte |

El token se firma con **RS256**, no con HS256, precisamente para esto: los demás
módulos reciben la clave **pública** y pueden verificar sin poder firmar. Con una
clave simétrica habría que repartirles la clave de firma, y cualquiera de los seis
equipos podría emitir un token de administrador.

Si vienes de otro equipo, lee [`specs/kit-integracion.md`](specs/kit-integracion.md):
es la guía práctica, con ejemplos ejecutables.

### Programa contra nosotros antes de que existamos

El contrato está congelado antes que el código. Levanta el entorno simulado:

```bash
npx @stoplight/prism-cli mock specs/openapi.yaml -p 4010
curl http://localhost:4010/.well-known/jwks.json
```

Responde con los ejemplos reales del contrato, incluidos los caminos de error.
Cuando la implementación esté lista, solo cambias la URL base.

## Documentación

Orden de lectura recomendado:

1. [`specs/overview.md`](specs/overview.md) — qué se construye y para quién.
2. [`specs/stack.md`](specs/stack.md) — con qué herramientas y por qué.
3. [`specs/api-contract.md`](specs/api-contract.md) — cómo hablan las dos apps.
4. [`specs/arquitectura.md`](specs/arquitectura.md) — diagramas de componentes.
5. [`specs/integracion.md`](specs/integracion.md) — cómo hablan los otros seis
   módulos con este. Si vienes de otro equipo, empieza por
   [`specs/kit-integracion.md`](specs/kit-integracion.md).
5. [`specs/registro.md`](specs/registro.md) — alta de cuentas y verificación de
   email, la funcionalidad en curso.
6. [`specs/login.md`](specs/login.md) — autenticación y gestión de sesiones.

Antes de tocar código, lee [`AGENTS.md`](AGENTS.md): convenciones, reglas de
alcance y cómo trabajar con los tres repositorios.

## Equipo

Equipo de 6 personas. El Product Owner aprueba las specs y resuelve las dudas
de alcance.
