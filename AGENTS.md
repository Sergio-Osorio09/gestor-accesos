# AGENTS.md

Guía operativa para cualquier agente de IA (o persona) que trabaje en este
repositorio. Léela completa antes de tocar un archivo.

## 1. Qué es este proyecto

`gestor-accesos` es el módulo de seguridad y autenticación de usuarios de un
marketplace. Cubre el registro, el inicio de sesión, la gestión de sesiones y
el control de permisos de los distintos actores de la plataforma.

El proyecto se desarrolla con metodología **SDD (Spec-Driven Development)**:
primero se escribe la especificación, después el código.

## 2. Dónde vive cada cosa

| Ruta                    | Contenido                                             |
| ----------------------- | ----------------------------------------------------- |
| `specs/overview.md`     | Problema, alcance, fuera de alcance y actores.         |
| `specs/stack.md`        | Tecnologías elegidas y su justificación.               |
| `specs/api-contract.md` | Contrato HTTP: convenciones y tabla de endpoints.      |
| `specs/arquitectura.md` | Diagrama de componentes y flujo de comunicación.       |
| `specs/login.md`        | Spec de la primera funcionalidad.                      |
| `specs/login.plan.md`   | Plan de implementación del login.                      |
| `specs/registro.md`     | Spec del alta de cuentas y verificación de email.      |
| `specs/registro.plan.md`| Plan de implementación del registro.                   |
| `specs/integracion.md`  | Spec de la API que consumen los otros seis módulos.    |
| `specs/integracion.plan.md` | Plan de implementación de la integración.          |
| `specs/kit-integracion.md`  | Guía práctica para los equipos consumidores.       |
| `specs/openapi.yaml`    | Contrato publicado a los demás equipos.                |
| `frontend/`             | SPA en React + TypeScript. **Repositorio Git aparte.** |
| `backend/`              | API en Spring WebFlux. **Repositorio Git aparte.**     |
| `AGENTS.md`             | Este documento.                                        |
| `CLAUDE.md`             | Puntero a este documento.                              |

La raíz es el **repositorio de specs**: es el repo del proyecto y contiene
únicamente documentación.

## 3. Tres repositorios Git independientes

`./frontend` y `./backend` son repositorios Git **independientes**, con su
propio historial y su propio `.gitignore`. El repo raíz los ignora a propósito.

Reglas innegociables:

- Todo comando de Git sobre el frontend o el backend se ejecuta **entrando a
  esa carpeta**: `cd frontend && git commit ...`. **Nunca desde la raíz.**
- Desde la raíz no se hace `git add frontend/` ni `git add backend/` ni
  `git add .` con la intención de incluirlos. Si aparecen en `git status` de
  la raíz, algo está mal: pará y avisá antes de commitear.
- No se usan submódulos. Los tres repos se clonan por separado (ver
  `README.md`).
- Un cambio que toca spec y código son **dos commits en dos repos distintos**.

## 4. La spec es la fuente de verdad

Si el código se desvía de la spec, **se corrige el código o se actualiza la
spec de forma explícita**; nunca se deja la divergencia sin registrar, y nunca
se reescribe la spec a posteriori para que "cuadre" con lo que ya se
implementó sin decirlo.

El orden correcto es: spec → visto bueno del Product Owner → código → tests.

## 5. Regla de alcance

**No se implementa nada que no esté en una spec.**

Si al implementar detectás un vacío, una ambigüedad o un caso no contemplado:
paras, lo preguntás al Product Owner y esperás respuesta. No se asume, no se
inventa comportamiento "razonable" ni se añaden funcionalidades extra por
iniciativa propia.

Tampoco se refactoriza código ajeno a la tarea en curso.

## 6. Convenciones de código

- **Idioma del código: inglés.** Nombres de variables, funciones, clases,
  archivos, ramas, endpoints y mensajes de commit en inglés.
- **Idioma de la documentación: español.** Todo lo que vive en `specs/` y los
  `.md` de la raíz.
- **Comentarios: solo donde la lógica no es obvia** — explicando el *por qué*,
  no el *qué*. Nada de comentarios decorativos, separadores ASCII, ni
  redundancias del tipo `// incrementa el contador`.
- **Tests: uno por cada criterio de aceptación de la spec.** Un criterio sin
  test se considera no implementado.
- Sin código muerto ni `TODO` sueltos: si algo queda pendiente, va a la spec.

## 7. Orden de lectura recomendado

1. `specs/overview.md` — qué se está construyendo y para quién.
2. `specs/stack.md` — con qué herramientas.
3. `specs/api-contract.md` — cómo hablan frontend y backend.
4. La spec de la funcionalidad en curso.

## 8. Equipo

Equipo de 6 personas. El **Product Owner** es quien aprueba specs y resuelve
las dudas de alcance. Ante cualquier decisión de producto, se le pregunta.
