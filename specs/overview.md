# Visión general

## 1. Problema que resuelve

Un marketplace pone en contacto a personas que no se conocen y que mueven
dinero entre ellas. Antes de poder comprar, vender o moderar, la plataforma
necesita responder con certeza a dos preguntas: **quién eres** y **qué se te
permite hacer**.

`gestor-accesos` es el módulo que responde a esas dos preguntas. Centraliza la
identidad y el control de acceso para que el resto de módulos del marketplace
(catálogo, pedidos, pagos, valoraciones) no tengan que resolverlo cada uno por
su cuenta, cada uno a su manera.

Sin este módulo, cada equipo reimplementaría su propia autenticación: es la
receta habitual para que aparezcan agujeros de seguridad inconsistentes y
difíciles de auditar.

## 2. Alcance

El módulo se responsabiliza de:

- **Registro** de cuentas nuevas y verificación de la dirección de email.
- **Autenticación** con email y contraseña.
- **Gestión de sesiones:** emisión, renovación y revocación de credenciales.
- **Recuperación de acceso:** restablecimiento de contraseña olvidada.
- **Modelo de roles:** a qué rol pertenece cada cuenta y qué permisos implica.
- **Protección frente a abuso:** política de intentos fallidos y bloqueo
  temporal de cuentas.
- **Estados de cuenta:** activa, desactivada, con email sin verificar,
  bloqueada temporalmente.

Cada punto se especifica en su propio documento antes de implementarse.
Especificados hasta ahora:

| Spec | Funcionalidad |
| --- | --- |
| [registro.md](registro.md) | Alta de cuentas y verificación de email |
| [login.md](login.md) | Autenticación, renovación y cierre de sesión |
| [integracion.md](integracion.md) | API de identidad para los demás módulos del marketplace |

El orden de implementación empieza por el registro: `login.md` asume cuentas que
ya existen, así que sin alta no hay nadie que pueda iniciar sesión.

## 3. Fuera de alcance

Lo siguiente **no** es responsabilidad de este módulo, aunque se relacione con
él:

- **La lógica de negocio del marketplace:** catálogo, carrito, pedidos, pagos,
  envíos, valoraciones. Este módulo solo dice quién eres; qué haces después es
  cosa de otros módulos.
- **Los datos de perfil no relacionados con el acceso:** dirección de envío,
  método de pago, preferencias, avatar. Aquí solo viven las credenciales, el
  estado de la cuenta y los roles.
- **La aplicación de los permisos en cada endpoint de negocio.** Este módulo
  emite los roles; cada módulo decide qué exige.
- **Los eventos asíncronos entre módulos.** Publicar en cola las bajas, los
  bloqueos y los cambios de rol es la vía 3 de [integracion.md](integracion.md),
  y se especifica aparte.
- **Identidades federadas** (Google, Apple, GitHub) y **SSO corporativo**.
- **Segundo factor de autenticación (MFA).**
- **Gestión de sesiones de administración interna** y herramientas de back
  office.
- **Infraestructura transversal:** WAF, protección DDoS, gestión de secretos y
  observabilidad, que pertenecen a la plataforma.

## 4. Actores del sistema

| Actor | Descripción | Qué hace en este módulo |
| --- | --- | --- |
| **Visitante** | Persona sin cuenta o sin sesión iniciada. | Se registra, inicia sesión, pide restablecer su contraseña. |
| **Comprador** (`BUYER`) | Usuario autenticado que compra. Es el rol por defecto al registrarse. | Inicia y cierra sesión, renueva su sesión, cambia su contraseña. |
| **Vendedor** (`SELLER`) | Usuario autenticado con permiso para publicar productos. | Lo mismo que el comprador; su rol se lo dan otros procesos. |
| **Administrador** (`ADMIN`) | Personal de la plataforma. | Desactiva y reactiva cuentas, desbloquea cuentas, consulta el registro de accesos. |
| **Módulos consumidores** | Los otros seis microservicios del marketplace. | Verifican el token de una petición con nuestra clave pública, y nos consultan por API lo que el token no lleva. **No pueden tocar nuestra base de datos.** Ver [integracion.md](integracion.md). |
| **Proveedor de correo** | Servicio externo de envío de emails. | Entrega los correos de verificación y de restablecimiento de contraseña. |

### Catálogo de roles

La tabla de actores nombra los tres roles que intervienen en las funcionalidades
ya especificadas. **No son todos los del marketplace:** el plan del curso define
seis, uno por perfil de módulo.

| Rol | Perfil |
| --- | --- |
| `BUYER` | Cliente que compra |
| `SELLER` | Vendedor que publica productos |
| `SALES_ADMIN` | Administra pedidos: anulaciones y reembolsos |
| `DISPATCH_MANAGER` | Administra rutas y entregas |
| `COMMERCIAL_MANAGER` | Administra catálogo y precios |
| `SYSTEM_ADMIN` | Personal de la plataforma |

**Estos códigos viajan dentro del token que leen los otros seis equipos**, y el
plan del curso los define en español (`CLIENTE`, `VENDEDOR`, …). La divergencia
está registrada en [`integracion.md` §5.10](integracion.md) y se acuerda en la
sincronización entre módulos **antes** de publicar el contrato.

El catálogo de permisos asociado a cada rol es su propia spec, todavía sin
escribir.

### Estados de una cuenta

Los estados que manejan todas las funcionalidades del módulo. Son **estados
conceptuales**, no valores de un único campo:

| Estado | Significado | Cómo se representa | Cómo se sale |
| --- | --- | --- | --- |
| Sin verificar | Registrada, email sin confirmar. | `emailVerified = false` | Verificando el email. |
| Activa | Operativa. | `status = ACTIVE`, `emailVerified = true`, sin bloqueo vigente | — |
| Bloqueada | Bloqueo temporal por intentos fallidos. | `lockedUntil` con un instante futuro | Sola, al expirar la ventana. |
| Desactivada | Desactivada por un administrador. | `status = DISABLED` | Solo un administrador la reactiva. |

**Los tres campos son independientes, y a propósito.** Un único enum no podría
representar las combinaciones que ocurren de verdad: una cuenta puede estar sin
verificar **y además** bloqueada por intentos fallidos, y un administrador puede
desactivar una cuenta que ya estaba bloqueada. Además el bloqueo necesita el
instante en que expira, que no cabe en un enum.

Cada funcionalidad declara en qué orden comprueba estos estados; para el login
está en [`login.md` §4.7](login.md).

## 5. Principios

1. **Los mensajes de error no filtran información.** Nunca se revela si un
   email está registrado a quien no conoce la credencial.
2. **Las contraseñas no se guardan, se hashean** (BCrypt), y nunca aparecen en
   logs ni en respuestas.
3. **Las credenciales de larga vida son revocables.** Las de corta vida se
   confían a su caducidad.
4. **Todo criterio de aceptación tiene un test.** Sin test, no está hecho.
