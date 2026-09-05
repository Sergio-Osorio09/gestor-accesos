# Registro de cuentas y verificación de email

## 1. Contexto

Da de alta cuentas nuevas en el marketplace y confirma que la dirección de email
pertenece de verdad a quien se registra. Es la puerta de entrada previa al
login: [`login.md`](login.md) asume cuentas que ya existen, y esta spec es la
que las crea.

## 2. Actores

| Actor | Descripción |
| --- | --- |
| **Visitante** | Persona sin cuenta. Rellena el formulario de alta y, después, abre el enlace del correo de verificación. |
| **Titular de un email ya registrado** | Persona con cuenta que recibe un aviso cuando alguien intenta registrarse con su dirección. No hace nada; solo se le informa. |
| **SPA (frontend React)** | Presenta el formulario, valida en el navegador antes de enviar y consume el token de verificación del enlace del correo. |
| **API de autenticación (Spring WebFlux)** | Valida los datos, crea la cuenta, genera el token de verificación y encarga el envío del correo. |
| **Proveedor de correo** | Servicio externo que entrega los mensajes. El flujo no se cierra sin él, pero su caída **no** impide crear la cuenta (ver 4.8). |

Los estados de cuenta y cómo se representan están definidos en
[`overview.md`](overview.md#estados-de-una-cuenta). Esta funcionalidad crea
cuentas con `emailVerified = false` y las pasa a `true` al verificarlas.

## 3. Escenarios

### Escenario 1 — Registro con datos válidos

**GIVEN** un visitante y una dirección de email que no corresponde a ninguna
cuenta
**WHEN** envía `POST /api/v1/auth/register` con email, contraseña y nombre
visible válidos
**THEN** la respuesta es `202` con un cuerpo genérico que **no** confirma si la
cuenta se creó
**AND** se crea la cuenta con `emailVerified = false`, `status = ACTIVE` y el
rol `BUYER`
**AND** la contraseña se guarda hasheada con BCrypt, nunca en claro
**AND** se genera un token de verificación de un solo uso, válido 24 horas
**AND** se encarga el envío del correo de verificación a esa dirección.

### Escenario 2 — Registro con un email ya registrado

**GIVEN** una dirección de email que **sí** corresponde a una cuenta existente
**WHEN** un visitante envía `POST /api/v1/auth/register` con esa dirección
**THEN** la respuesta es `202` y su cuerpo es **idéntico, byte a byte**, al del
escenario 1
**AND** no se crea ninguna cuenta, no se modifica la existente y no se toca su
contraseña
**AND** se envía al titular un correo de aviso distinto: alguien intentó
registrarse con su dirección, con enlaces al login y a recuperar contraseña.

> Los escenarios 1 y 2 son indistinguibles desde fuera **a propósito**. Es la
> misma postura anti-enumeración que toma `login.md` en sus escenarios 2 y 3: si
> el formulario de alta dijera "ese email ya está en uso", cualquiera podría
> averiguar qué direcciones tienen cuenta en el marketplace.

### Escenario 3 — Email con formato inválido

**GIVEN** un visitante en el formulario de alta
**WHEN** envía un email sin `@`, sin dominio o de más de 254 caracteres
**THEN** la respuesta es `400` con `code: VALIDATION_ERROR` y un array `errors`
que señala el campo `email`
**AND** no se crea nada ni se envía ningún correo.

### Escenario 4 — Contraseña demasiado corta

**GIVEN** un visitante en el formulario de alta
**WHEN** envía una contraseña de menos de 12 caracteres
**THEN** la respuesta es `400` con `code: WEAK_PASSWORD`
**AND** el mensaje indica el mínimo, sin exigir mayúsculas, dígitos ni símbolos.

### Escenario 5 — Contraseña en la lista de comunes

**GIVEN** un visitante en el formulario de alta
**WHEN** envía una contraseña que figura en la lista de contraseñas más usadas
(por ejemplo `contrasena123`)
**THEN** la respuesta es `400` con `code: WEAK_PASSWORD`
**AND** el mensaje explica que es una contraseña demasiado común.

### Escenario 6 — Contraseña de más de 72 bytes

**GIVEN** un visitante en el formulario de alta
**WHEN** envía una contraseña cuya codificación UTF-8 supera los 72 bytes
**THEN** la respuesta es `400` con `code: WEAK_PASSWORD`
**AND** la contraseña se rechaza en lugar de truncarse (ver 4.4).

### Escenario 7 — Nombre visible fuera de los límites

**GIVEN** un visitante en el formulario de alta
**WHEN** envía un nombre visible vacío, de un solo carácter o de más de 100
**THEN** la respuesta es `400` con `code: VALIDATION_ERROR` señalando el campo
`displayName`.

### Escenario 8 — Verificación con un token válido

**GIVEN** una cuenta recién creada con `emailVerified = false` y un token de
verificación sin usar y sin caducar
**WHEN** se envía `POST /api/v1/auth/verify-email` con ese token
**THEN** la respuesta es `200` con los datos públicos de la cuenta
**AND** la cuenta pasa a `emailVerified = true`
**AND** el token queda consumido y no sirve una segunda vez.

### Escenario 9 — Verificación con un token caducado

**GIVEN** un token de verificación emitido hace más de 24 horas
**WHEN** se envía `POST /api/v1/auth/verify-email` con él
**THEN** la respuesta es `410` con `code: VERIFICATION_TOKEN_EXPIRED`
**AND** la cuenta sigue sin verificar
**AND** el mensaje invita a pedir un reenvío.

### Escenario 10 — Verificación con un token ya usado

**GIVEN** un token de verificación que ya se consumió en el escenario 8
**WHEN** se envía `POST /api/v1/auth/verify-email` con él otra vez
**THEN** la respuesta es `410` con `code: VERIFICATION_TOKEN_INVALID`
**AND** la cuenta permanece verificada: reutilizar el token no la deshace.

### Escenario 11 — Reenvío del correo de verificación

**GIVEN** una cuenta sin verificar cuyo último correo se envió hace más de 60
segundos
**WHEN** se envía `POST /api/v1/auth/resend-verification` con su email
**THEN** la respuesta es `202` con un cuerpo genérico
**AND** se invalida el token anterior y se emite uno nuevo con 24 horas de vida
**AND** se encarga el envío de un correo de verificación nuevo.

### Escenario 12 — Login inmediatamente después de verificar

**GIVEN** una cuenta que acaba de completar el escenario 8
**WHEN** el usuario envía `POST /api/v1/auth/login` con su email y su contraseña
**THEN** la respuesta es `200` con access token y cookie de refresh, según
`login.md`
**AND** ya no se responde `403 EMAIL_NOT_VERIFIED`.

## 4. Casos límite

### 4.1 Normalización del email

El email se guarda y se compara en minúsculas y sin espacios en los extremos,
igual que en [`login.md` §4.2](login.md). Así una cuenta registrada como
`Ada@Example.com` inicia sesión con `ada@example.com`.

### 4.2 Alias y puntos en la dirección

`ada+tienda@gmail.com`, `ada@gmail.com` y `a.d.a@gmail.com` se tratan como
**tres direcciones distintas**. No se intenta canonizar alias ni quitar puntos:
esas reglas son propias de cada proveedor de correo, cambian sin avisar, y
adivinarlas mal fusionaría cuentas de personas distintas.

Se acepta el coste: alguien puede crear varias cuentas con alias de un mismo
buzón. Mitigarlo es cosa del anti-abuso, que está fuera de alcance.

### 4.3 Dos registros simultáneos con el mismo email

La unicidad la garantiza la **restricción `UNIQUE` de la base de datos**, no una
comprobación previa. Consultar y luego insertar deja una ventana de carrera en
la que dos peticiones simultáneas pasan las dos la comprobación.

Cuando la inserción viola la restricción, la petición perdedora se desvía al
mismo camino del escenario 2: responde `202` y avisa por correo al titular.

### 4.4 Límites de la contraseña

- **Mínimo 12 caracteres.** Sin exigir mayúsculas, dígitos ni símbolos: las
  reglas de composición empujan a patrones predecibles (`Password1!`) y no
  aportan seguridad real.
- **Máximo 72 bytes UTF-8**, el límite de BCrypt, que trunca en silencio a
  partir de ahí. Se rechaza en lugar de truncar, porque aceptarla daría una
  falsa sensación de seguridad. El límite es en **bytes**: los acentos y los
  emojis ocupan más de uno.
- **No se normaliza** la contraseña: ni espacios, ni mayúsculas, ni forma
  Unicode. Solo el email y el nombre visible se normalizan.

### 4.5 Contraseñas comunes o derivadas de los datos de la cuenta

Se rechaza con `WEAK_PASSWORD` una contraseña que:

- figure en la lista de contraseñas más filtradas que se empaqueta con la
  aplicación;
- coincida con el email, con la parte local del email o con el nombre visible;
- sea una de esas cadenas con dígitos añadidos al final.

La comprobación es local: **no se consulta ningún servicio externo** ni se envía
la contraseña ni su hash a terceros.

### 4.6 Token de verificación

Valor aleatorio de 256 bits, del que en la base de datos se guarda **solo su
SHA-256**. Una filtración de la tabla no permite verificar cuentas ajenas.

- **Un solo uso:** se consume al verificar.
- **24 horas de vida.**
- Un reenvío **invalida el token anterior**: solo el último vale.
- Verificar una cuenta que ya está verificada responde `410`, no `200`: el token
  ya se consumió y no hay nada que hacer.
- Un token inexistente y uno ya usado responden lo mismo,
  `VERIFICATION_TOKEN_INVALID`, para no revelar cuáles existieron.

### 4.7 Reenvío: cooldown y anti-enumeración

Cooldown de **60 segundos por cuenta**. Pedirlo antes responde `429` con
`code: RESEND_TOO_SOON` y `retryAfterSeconds`.

El reenvío responde `202` **aunque el email no corresponda a ninguna cuenta o
la cuenta ya esté verificada**, y el cuerpo es idéntico en los tres casos. Si
distinguiera, sería otra vía de enumeración, y quedaría abierta la que el
escenario 2 cierra.

### 4.8 Caída del proveedor de correo

Si el envío falla, **la cuenta se crea igual** y la respuesta sigue siendo
`202`. El envío se encola y se reintenta por separado.

Atar la creación de la cuenta al envío del correo significaría perder altas cada
vez que un tercero tenga una mala tarde. El usuario que no recibe el correo
tiene salida: el reenvío del escenario 11.

### 4.9 Registro sobre el email de una cuenta desactivada

Una cuenta en `DISABLED` sigue siendo una cuenta existente: se aplica el
escenario 2. **No se reactiva, no se sobrescribe la contraseña y no se crea una
cuenta paralela.** Solo un administrador reactiva una cuenta desactivada, y eso
está fuera de alcance.

### 4.10 Unicode en el nombre visible

Se normaliza a **NFC** y se recortan los espacios de los extremos. Se rechazan
los caracteres de control y los espacios de ancho cero, que sirven para
suplantar nombres ajenos visualmente.

No se exige unicidad: dos personas pueden llamarse igual. El identificador único
es el email.

### 4.11 Orden de las comprobaciones

1. Formato del cuerpo — email, nombre visible → `400 VALIDATION_ERROR`
2. Reglas de la contraseña → `400 WEAK_PASSWORD`
3. Existencia de la cuenta → **nunca produce error**: escenario 1 o 2

Las validaciones van **antes** de mirar si el email existe. Así una petición mal
formada recibe el mismo `400` exista o no la cuenta, y el orden de las
comprobaciones no se convierte en un canal para deducirlo.

## 5. Fuera de alcance

Nada de lo siguiente se implementa en esta funcionalidad. Si hace falta, se
especifica aparte.

- **Inicio de sesión, renovación y cierre de sesión:** es [`login.md`](login.md).
- **Recuperación de contraseña** ("olvidé mi contraseña") y **cambio de
  contraseña** estando autenticado.
- **CAPTCHA, anti-bot y rate limiting por IP.** El único límite aquí es el
  cooldown de reenvío por cuenta (4.7).
- **Registro por invitación** o con código de referido.
- **Alta de vendedores con verificación de identidad (KYC).** Todas las cuentas
  nacen con rol `BUYER`; **no se elige rol al registrarse**.
- **Registro federado** (Google, Apple, GitHub).
- **Aceptación versionada de términos y condiciones** y registro del
  consentimiento de privacidad.
- **Cambio de la dirección de email** de una cuenta ya creada.
- **Baja y borrado de cuenta**, y el borrado de las cuentas que nunca se
  verifican.
- **Plantillas HTML de los correos**, su diseño y su traducción. Esta spec fija
  *qué* correo se envía y *cuándo*, no cómo se ve.
- **Verificación por SMS o teléfono.**
- **Gestión de rebotes:** qué hacer si el correo de verificación no se entrega.
- **Internacionalización** de los mensajes de error: solo español.
