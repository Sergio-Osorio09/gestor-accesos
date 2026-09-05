# API de identidad para los demás módulos

## 1. Contexto

El marketplace lo construyen siete equipos. Seis de ellos —productos y ofertas,
ventas y postventa, despacho y entrega, chatbot, retail y administración— tienen
que saber **quién** está detrás de cada petición y **qué se le permite hacer**,
y ninguno puede consultar nuestra base de datos: la matriz de integración del
curso lo prohíbe expresamente. Toda relación pasa por API.

Esa prohibición no es burocracia. Si el módulo de despacho leyera nuestra tabla
`users`, cualquier cambio de esquema nuestro rompería su despliegue, y una
consulta suya mal escrita filtraría hashes de contraseña. La API es la frontera
que hace que ese acoplamiento no exista.

Esta spec define esa frontera. Es la única que bloquea a equipos que no son el
nuestro: mientras no exista, los otros seis no pueden ni empezar a programar sus
llamadas. Por eso se escribe y se publica **antes** que la implementación, y no
después.

Las funcionalidades internas del módulo están en [`registro.md`](registro.md) y
[`login.md`](login.md). Esta spec no las repite: describe lo que ocurre **desde
fuera**, cuando quien llama no es nuestra SPA sino otro microservicio.

## 2. Actores

| Actor | Descripción |
| --- | --- |
| **Módulo consumidor** | Uno de los otros seis microservicios del marketplace. Recibe peticiones de usuarios finales que traen un token emitido por nosotros, y a veces necesita preguntarnos por datos que el token no lleva. |
| **Usuario final** | Comprador, vendedor o administrador. No habla con esta API directamente: su token viaja dentro de las peticiones que hace al módulo consumidor. |
| **API de identidad (este módulo)** | Emite los tokens, publica la clave con la que se verifican y responde a las consultas de los módulos consumidores. |
| **Equipo consumidor** | Las personas del otro equipo. Necesitan el contrato, sus credenciales de servicio y un entorno contra el que programar antes de que exista nuestra implementación. |

Un módulo consumidor tiene **dos identidades distintas** frente a nosotros, y
confundirlas es el error más caro de esta integración:

- **La identidad del usuario final**, que viaja en el token que el módulo recibe
  y que él solo **verifica**; nunca la emite ni la modifica.
- **Su propia identidad de servicio**, con la que se autentica cuando *él* nos
  llama. Es un `client_id` propio de cada módulo, para que la auditoría diga qué
  módulo consultó qué dato.

## 3. Las tres vías de comunicación

Antes de los escenarios, el mapa. Un módulo consumidor se relaciona con nosotros
de tres formas, y la primera cubre la inmensa mayoría del tráfico.

| Vía | Cuándo | Coste |
| --- | --- | --- |
| **1. Validación local del token** | En cada petición ordinaria que recibe el módulo | Ninguna llamada de red. No detecta cambios de estado hasta que el token vence |
| **2. Llamada síncrona a nuestra API** | Antes de operaciones sensibles, o para datos que el token no lleva | Una llamada de red y una dependencia de nuestra disponibilidad |
| **3. Eventos asíncronos** | Para enterarse de bajas y bloqueos sin preguntar | **Fuera de alcance de esta spec** (bloque 5) |

**La vía 1 es la que hay que usar por defecto.** Un módulo que llame a nuestra
introspección en cada petición nos convierte en su punto único de fallo: si
nosotros caemos, cae el marketplace entero. La vía 2 existe para el puñado de
operaciones en las que 15 minutos de desfase son inaceptables.

## 4. Escenarios

### Escenario 1 — Validación local del token con JWKS

**GIVEN** el módulo de productos recibe una petición con
`Authorization: Bearer <token>` emitido por nosotros
**WHEN** obtiene la clave pública de `GET /api/v1/.well-known/jwks.json`,
selecciona la clave cuyo `kid` coincide con el de la cabecera del token y
verifica la firma
**THEN** confirma la validez del token y lee `sub`, `roles` y `permissions`
**AND** no realiza **ninguna** llamada a este módulo
**AND** el resultado es el mismo aunque nuestra API esté caída, mientras el JWKS
siga en su caché.

### Escenario 2 — Obtención de un token de servicio

**GIVEN** el equipo de despacho tiene un `client_id` y un `client_secret`
propios
**WHEN** envía `POST /api/v1/auth/token` con `grant_type=client_credentials`
**THEN** la respuesta es `200` con `accessToken`, `tokenType: "Bearer"`,
`expiresIn` y la lista de `scopes` concedidos a ese cliente
**AND** el token emitido lleva `type: "service"` y el `client_id` en `sub`, no
un identificador de usuario
**AND** los scopes son exactamente los concedidos a ese módulo, aunque haya
pedido más.

### Escenario 3 — Introspección de un token de usuario válido

**GIVEN** el módulo de ventas va a anular un pedido y necesita el estado
**actual** de quien lo pide
**WHEN** envía `POST /api/v1/auth/introspect` con el token del usuario y su
propio token de servicio con scope `tokens:introspect`
**THEN** la respuesta es `200` con `active: true`, el `sub`, los `roles` y los
`permissions` vigentes **en este instante**, no los que llevaba el token
**AND** la respuesta refleja cualquier cambio de rol posterior a la emisión.

### Escenario 4 — Introspección de un token de un usuario desactivado

**GIVEN** un usuario fue desactivado hace 3 minutos y su token de acceso, emitido
antes, aún no vence
**WHEN** un módulo lo introspecciona
**THEN** la respuesta es `200` con `active: false` y `reason: "USER_INACTIVE"`
**AND** la firma del token sigue siendo criptográficamente válida: lo que dice
la respuesta es que **no es utilizable**, no que esté mal firmado
**AND** el módulo consumidor debe negar la operación.

> Este escenario es la razón de ser de la vía 2. La vía 1 no puede detectar esto:
> la firma es correcta y el `exp` no ha llegado.

### Escenario 5 — Consulta de datos básicos de un usuario

**GIVEN** el módulo de ventas necesita mostrar el nombre del cliente de un pedido
**WHEN** envía `GET /api/v1/users/{id}` con un token de servicio con scope
`users:read`
**THEN** la respuesta es `200` con `id`, `displayName`, `email`, `status`,
`emailVerified` y `roles`
**AND** **no** incluye el hash de la contraseña, ni el documento, ni ningún dato
que su scope no autorice.

### Escenario 6 — Consulta por lote con identificadores inexistentes

**GIVEN** el módulo de despacho necesita los datos de 10 usuarios, y 2 de esos
identificadores no corresponden a ninguna cuenta
**WHEN** envía `POST /api/v1/users/batch` con los 10 identificadores
**THEN** la respuesta es `200` con los 8 usuarios encontrados en `users` y los 2
identificadores no hallados en `notFound`
**AND** la petición **no falla entera** por los 2 que faltan.

> Que un `404` parcial no tumbe la consulta completa no es comodidad: un reporte
> de 50 clientes en el que uno se dio de baja seguiría funcionando, en vez de
> quedarse en blanco.

### Escenario 7 — Consulta de direcciones por el módulo de despacho

**GIVEN** el módulo de despacho necesita la dirección de entrega de un pedido
**WHEN** envía `GET /api/v1/users/{id}/addresses` con scope `addresses:read`
**THEN** la respuesta es `200` con las direcciones del cliente
**AND** **no** incluye el número de documento ni la fecha de nacimiento, que no
hacen falta para entregar un paquete.

### Escenario 8 — Dato sensible con el scope que lo autoriza

**GIVEN** el módulo de ventas debe emitir una boleta y necesita el documento del
cliente
**WHEN** consulta los datos de facturación con un token de servicio con scope
`users:read:document`
**THEN** la respuesta incluye el nombre completo y el número de documento **en
claro**
**AND** el acceso queda registrado en auditoría con el módulo solicitante, el
identificador consultado y la fecha.

### Escenario 9 — Dato sensible sin el scope que lo autoriza

**GIVEN** el módulo de chatbot consulta los datos de un cliente y **no** tiene el
scope `users:read:document`
**WHEN** pide los datos de ese cliente
**THEN** la respuesta es `200` con el documento **enmascarado**, mostrando solo
los últimos tres dígitos
**AND** la petición no se rechaza: se responde con menos, no con un error.

> Se enmascara en vez de devolver `403` porque el chatbot sí necesita confirmar
> ante el cliente los últimos dígitos de su documento. Un `403` le obligaría a
> pedir el scope completo para un caso que no lo justifica.

### Escenario 10 — Petición sin token de servicio

**GIVEN** un módulo consulta los datos básicos de un usuario
**WHEN** no incluye ninguna cabecera `Authorization`
**THEN** la respuesta es `401` con `code: TOKEN_INVALID`
**AND** el cuerpo es **idéntico** exista o no el usuario consultado: un `401` no
puede convertirse en una forma de averiguar qué identificadores existen.

### Escenario 11 — Petición con scope insuficiente

**GIVEN** el módulo de chatbot presenta un token de servicio válido
**WHEN** invoca un endpoint de administración de roles, para el que no tiene
scope
**THEN** la respuesta es `403` con `code: INSUFFICIENT_SCOPE`
**AND** el intento queda registrado en auditoría como evento de seguridad
**AND** la respuesta no revela qué scope habría hecho falta.

### Escenario 12 — Consulta del catálogo de roles

**GIVEN** un módulo consumidor necesita saber qué significa el rol
`DISPATCH_MANAGER` que le llegó en un token
**WHEN** envía `GET /api/v1/roles` con scope `roles:read`
**THEN** la respuesta es `200` con cada rol, su descripción y sus permisos
**AND** el módulo puede resolver los permisos sin consultar nuestra base de
datos ni codificarlos a mano.

### Escenario 13 — Rotación de la clave de firma

**GIVEN** hay tokens en circulación firmados con la clave `kid: "2026-09"`
**WHEN** se rota la clave de firma y se empieza a emitir con `kid: "2026-10"`
**THEN** el JWKS publica **las dos** claves
**AND** la clave antigua se mantiene publicada al menos 24 horas
**AND** ningún token en circulación deja de validarse durante la rotación.

## 5. Casos límite

### 5.1 La ventana de incoherencia de 15 minutos

Un token de acceso sigue siendo válido hasta su `exp` aunque el usuario haya
sido desactivado, bloqueado o le hayan cambiado los roles. Con 15 minutos de
vida, esa es la ventana máxima de desfase de la vía 1.

**Es una consecuencia deliberada de firmar con RS256 y no una carencia.** El
precio de que los seis módulos validen sin llamarnos es que no se enteran de un
cambio hasta que el token vence. Quien no pueda pagarlo —anulaciones,
reembolsos, cambios de precio— usa la introspección de la vía 2. Reducir la vida
del token multiplicaría las renovaciones sin cerrar la ventana del todo.

### 5.2 Disponibilidad del JWKS

El JWKS debe cachearse en el módulo consumidor, no descargarse por petición. Un
consumidor que lo pida en cada validación anula la ventaja de la vía 1 y nos
convierte igualmente en su punto único de fallo.

Si el JWKS no responde y el consumidor tiene una copia en caché, **debe seguir
validando con ella**. Solo si llega un token con un `kid` desconocido y el JWKS
no responde, la petición se rechaza.

### 5.3 Límites de la consulta por lote

- Máximo **100 identificadores** por llamada. Más de 100 responde `400` con
  `code: BATCH_TOO_LARGE`; no se trunca en silencio.
- Los identificadores duplicados se colapsan: el mismo usuario aparece una sola
  vez en la respuesta.
- Un identificador malformado —que no sea un UUID— es un `400`
  `VALIDATION_ERROR`, no un "no encontrado". Un UUID bien formado que no existe
  sí va a `notFound`.

### 5.4 Anti-enumeración desde otros módulos

Las respuestas a un módulo consumidor no pueden convertirse en un canal para
averiguar qué cuentas existen. En concreto:

- Un `401` por token de servicio ausente o inválido es idéntico exista o no el
  usuario consultado, y se decide **antes** de mirar si existe.
- Un `403` por scope insuficiente tampoco depende de la existencia del usuario.
- Solo con un token de servicio válido y el scope correcto se llega a distinguir
  "existe" de "no existe".

Es la misma postura de [`login.md`](login.md) y [`registro.md`](registro.md),
aplicada a la frontera entre módulos.

### 5.5 Mínima exposición por scope

Cada consumidor recibe **solo los campos que su scope autoriza**, y el conjunto
de scopes de cada módulo se concede por escrito, no por petición. Un módulo que
necesite un campo nuevo lo pide en la sincronización entre equipos; no se
concede sobre la marcha.

Los scopes definidos por esta spec:

| Scope | Qué autoriza |
| --- | --- |
| `tokens:introspect` | Introspección de tokens de usuario |
| `users:read` | Datos básicos de usuario, individuales y por lote |
| `users:read:document` | El número de documento en claro en vez de enmascarado |
| `addresses:read` | Direcciones de entrega del cliente |
| `roles:read` | Catálogo de roles y permisos |

### 5.6 El token de servicio no es un token de usuario

Un token de servicio identifica a un **módulo**, no a una persona. En concreto:

- No sirve para invocar endpoints que actúan en nombre de un usuario
  (`/auth/me`, cambio de contraseña, cierre de sesión).
- No hereda los permisos del usuario cuyo token está introspeccionando.
- Un módulo **no puede** emitir tokens de usuario, ni obtenerlos por
  suplantación. Si lo necesita, el flujo pasa por el usuario final.

### 5.7 Caducidad y revocación de un token de servicio

El token de servicio caduca y se renueva pidiendo otro con las mismas
credenciales. Si un `client_secret` se filtra, se revoca **ese** cliente sin
afectar a los otros cinco módulos: es la razón de dar credenciales por módulo y
no una clave compartida.

### 5.8 Cambio incompatible del contrato

Si hace falta renombrar un campo que ya consumen otros módulos, **no se
renombra**: se añade el campo nuevo manteniendo el anterior, se comunica la
fecha de retirada en el canal de integración con al menos una semana de
antelación, y el campo antiguo se elimina solo tras confirmación de todos los
módulos afectados.

Un cambio que no admita ese periodo de convivencia obliga a `/api/v2`, con las
dos versiones conviviendo hasta que los consumidores migren.

### 5.9 Auditoría del acceso entre módulos

Queda registrado, como mínimo: el módulo solicitante, el endpoint, el
identificador consultado, si el dato sensible se entregó en claro o enmascarado,
y la fecha. Un `403` por scope insuficiente se registra como evento de
seguridad.

### 5.10 Códigos de rol pendientes de acuerdo

Esta spec usa los códigos de rol en inglés, coherentes con
[`AGENTS.md`](../AGENTS.md) §6: `BUYER`, `SELLER`, `SALES_ADMIN`,
`DISPATCH_MANAGER`, `COMMERCIAL_MANAGER`, `SYSTEM_ADMIN`.

**El plan del curso los define en español** (`CLIENTE`, `VENDEDOR`,
`ADMIN_VENTAS`, `GESTOR_DESPACHO`, `GESTOR_COMERCIAL`, `ADMIN_SISTEMA`), y esos
códigos **viajan dentro del token** que leen los otros seis equipos. La
divergencia se acuerda en la sincronización entre módulos **antes** de publicar
el contrato; hasta entonces queda registrada aquí en vez de resolverse por
nuestra cuenta.

### 5.11 Dependencia del catálogo de permisos

El claim `permissions` presupone un catálogo de permisos por módulo
(`PRODUCT_CREATE`, `ORDER_CANCEL`…) que **todavía no está especificado** en este
repositorio. Esta spec fija el **formato** —lista de códigos, sin duplicados,
unión de los roles del usuario— y declara la dependencia; el catálogo lo define
la spec de roles y permisos.

Hasta que exista, `permissions` viaja como lista vacía y los consumidores
autorizan por `roles`.

## 6. Requisitos no funcionales

- La introspección responde en menos de **200 ms** en el percentil 95.
- La consulta por lote acepta hasta 100 identificadores y responde en menos de
  **1 segundo**.
- El JWKS es cacheable y tolera la rotación manteniendo la clave anterior al
  menos **24 horas**.
- La documentación OpenAPI se genera desde el código, para que no se
  desincronice de la implementación.
- Existe un **entorno simulado** con datos de prueba y credenciales conocidas,
  disponible antes de que exista la implementación definitiva, para que los
  equipos consumidores no queden bloqueados.

## 7. Fuera de alcance

Nada de lo siguiente se implementa en esta funcionalidad. Si hace falta, se
especifica aparte.

- **Los eventos asíncronos.** La publicación en cola de `user.deactivated`,
  `user.locked`, `user.roles_changed`, `user.created` y
  `user.attributes_updated` es la vía 3, y se especifica por separado. Hasta que
  exista, la ventana de incoherencia es la del punto 5.1 y quien no la tolere usa
  la introspección.
- **La autorización de las operaciones de negocio.** Entregamos identidad, roles
  y permisos; qué permite hacer cada módulo con ellos lo decide ese módulo.
- **El catálogo de roles y permisos**, que es su propia spec (ver 5.11).
- **Los atributos de perfil y las direcciones**, cuyo modelo lo define la spec de
  atributos de usuario. Aquí solo se especifica **cómo se consultan desde otro
  módulo**, no cómo se crean ni se validan.
- **La pasarela de API centralizada** del marketplace y la infraestructura de red
  compartida.
- **Limitación de tasa por módulo consumidor.** Un consumidor mal programado
  puede saturarnos; mitigarlo es anti-abuso, y está fuera de alcance.
- **Federación de identidad** con proveedores externos.
- **El aprovisionamiento de los clientes de servicio.** Esta spec asume que cada
  módulo tiene su `client_id` y su `client_secret`; cómo se crean, se entregan y
  se rotan es trabajo de operaciones.
