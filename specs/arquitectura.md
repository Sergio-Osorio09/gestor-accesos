# Arquitectura

## 1. Vista general

Dos aplicaciones desplegadas por separado, comunicadas solo por HTTP siguiendo
[api-contract.md](api-contract.md). No comparten código ni base de datos.

```mermaid
graph TB
    subgraph navegador["🌐 Navegador"]
        SPA["React SPA<br/>TypeScript + Vite<br/><br/>Access token en memoria"]
        COOKIE[("Cookie httpOnly<br/>refresh_token<br/><br/>Inaccesible desde JS")]
    end

    subgraph servidor["☕ Backend — Spring Boot 4 + WebFlux"]
        API["AuthController<br/>/api/v1/auth/*"]
        SEC["SecurityWebFilterChain<br/>Valida el Bearer token"]
        UC["Casos de uso<br/>Login · Refresh · Logout"]
        JWT["JwtTokenService<br/>Firma y verifica HS256"]
        BCRYPT["PasswordHasher<br/>BCrypt en boundedElastic"]
        REPO["Repositorios R2DBC<br/>No bloqueantes"]
    end

    DB[("🐘 PostgreSQL 16<br/><br/>users · user_roles<br/>refresh_tokens")]

    SPA -->|"HTTPS + JSON<br/>Authorization: Bearer"| SEC
    COOKIE -.->|"Se adjunta sola<br/>solo en /api/v1/auth"| SEC
    SEC --> API
    API --> UC
    UC --> JWT
    UC --> BCRYPT
    UC --> REPO
    REPO -->|"R2DBC<br/>reactivo"| DB
    SPA -.->|"El navegador la guarda<br/>JS nunca la lee"| COOKIE

    classDef front fill:#61dafb,stroke:#1a7f9c,color:#000
    classDef back fill:#6db33f,stroke:#3d6b21,color:#fff
    classDef data fill:#336791,stroke:#1a3d5c,color:#fff
    class SPA front
    class COOKIE data
    class API,SEC,UC,JWT,BCRYPT,REPO back
    class DB data
```

## 2. Flujo de una autenticación

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuario
    participant S as React SPA
    participant A as API WebFlux
    participant D as PostgreSQL

    U->>S: email + contraseña
    S->>A: POST /api/v1/auth/login
    A->>D: Busca el usuario por email
    D-->>A: Usuario + hash + estado

    Note over A: Orden fijo de comprobaciones:<br/>bloqueo → contraseña →<br/>verificación → estado

    alt Credenciales correctas y cuenta operativa
        A->>D: Reinicia el contador de fallos
        A->>D: Guarda el refresh token (SHA-256)
        A-->>S: 200 + accessToken (15 min)
        Note over S,A: Set-Cookie: refresh_token<br/>HttpOnly · Secure · SameSite=Strict
        S->>S: Guarda el token EN MEMORIA
        S-->>U: Sesión iniciada
    else Credenciales incorrectas
        A->>D: Incrementa el contador de fallos
        A-->>S: 401 INVALID_CREDENTIALS
        S-->>U: "Email o contraseña incorrectos"
    else Quinto fallo consecutivo
        A->>D: locked_until = ahora + 15 min
        A-->>S: 423 ACCOUNT_LOCKED
        S-->>U: "Cuenta bloqueada 15 minutos"
    end
```

## 3. Renovación transparente de la sesión

Cuando el access token caduca, la SPA lo renueva sin que el usuario se entere:

```mermaid
sequenceDiagram
    autonumber
    participant S as React SPA
    participant A as API WebFlux
    participant D as PostgreSQL

    S->>A: GET /api/v1/... (token caducado)
    A-->>S: 401 TOKEN_EXPIRED
    S->>A: POST /api/v1/auth/refresh (con la cookie)
    A->>D: Busca por SHA-256 del token

    alt Token válido y sin usar
        A->>D: Revoca el antiguo, guarda el nuevo
        A-->>S: 200 + accessToken nuevo + cookie nueva
        S->>A: Reintenta la petición original
        A-->>S: 200
    else Token ya rotado — posible robo
        A->>D: Revoca TODA la familia del usuario
        A-->>S: 401 TOKEN_INVALID
        S->>S: Descarta la sesión y va al login
    end
```

## 4. Decisiones estructurales

**Dos despliegues, no uno.** La SPA es estática (se sirve desde un CDN o un
nginx); la API es un proceso Java. Escalan por separado y se despliegan por
separado. En desarrollo, el proxy de Vite redirige `/api` al backend para
evitar CORS.

**El backend no guarda estado de sesión en memoria.** El bloqueo de cuentas y
los refresh tokens viven en PostgreSQL. Así se puede levantar más de una
instancia detrás de un balanceador sin sesiones pegajosas, y un reinicio no
borra un bloqueo en curso.

**Organización por funcionalidad, no por capa técnica.** El paquete `auth`
contiene su API, sus casos de uso, su dominio y su infraestructura. Con 6
personas trabajando en paralelo, esto reduce los conflictos de merge frente a
un `controllers/` compartido por todos.

**El dominio no conoce a Spring.** Las entidades y los puertos de repositorio
son Java puro; los adaptadores R2DBC implementan esos puertos. Los casos de uso
se prueban sin levantar el contexto de Spring.
