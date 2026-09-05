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

Cada punto se especifica en su propio documento antes de implementarse. El
primero es [login.md](login.md).

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
| **Servicios internos** | Otros módulos del marketplace. | Validan el token de una petición y leen la identidad y los roles que contiene. |
| **Proveedor de correo** | Servicio externo de envío de emails. | Entrega los correos de verificación y de restablecimiento de contraseña. |

### Estados de una cuenta

Los estados que manejan todas las funcionalidades del módulo:

| Estado | Significado | Cómo se sale |
| --- | --- | --- |
| `PENDING_VERIFICATION` | Registrada, email sin verificar. | Verificando el email. |
| `ACTIVE` | Operativa. | — |
| `LOCKED` | Bloqueo temporal por intentos fallidos. | Solo, al expirar la ventana. |
| `DISABLED` | Desactivada por un administrador. | Solo un administrador la reactiva. |

## 5. Principios

1. **Los mensajes de error no filtran información.** Nunca se revela si un
   email está registrado a quien no conoce la credencial.
2. **Las contraseñas no se guardan, se hashean** (BCrypt), y nunca aparecen en
   logs ni en respuestas.
3. **Las credenciales de larga vida son revocables.** Las de corta vida se
   confían a su caducidad.
4. **Todo criterio de aceptación tiene un test.** Sin test, no está hecho.
