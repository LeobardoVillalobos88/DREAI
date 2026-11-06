# AGENTS.md — DREA

Instrucciones para agentes de código que trabajen en este repositorio.

**DREA** (Documentación de Requerimientos y Extracción Automatizada) es una plataforma web que extrae requerimientos de software desde documentos y audio con Google Gemini, los organiza en sprints sobre un tablero Kanban y genera el Documento Formal de Requerimientos (DFR) en PDF.

## Estado del repositorio

Por ahora solo hay especificación: aún no existe código, dependencias ni comandos. **No inventes comandos de build, test o lint.** Al crear el scaffolding, agrega aquí la estructura de carpetas y los comandos reales.

## Fuentes de verdad

1. [PRD.md](PRD.md): roles, pantallas, modelo de datos y criterios de aceptación (CA-01 a CA-08). Tiene prioridad.
2. [BRIEF.md](BRIEF.md): alcance y decisiones de producto del MVP.

La metodología es Spec-Driven Development: el código sigue a la especificación.

- Consulta la sección correspondiente del PRD antes de implementar; este archivo no la reemplaza.
- Si la especificación es ambigua o se contradice, pregunta antes de decidir. No rellenes huecos con supuestos propios.
- Si un cambio altera el comportamiento especificado, actualiza el PRD en el mismo cambio.

## Stack

| Capa | Tecnología |
|---|---|
| Backend | Django, autenticación JWT, modelo `auth.User` con `PerfilUsuario` |
| Base de datos | MySQL |
| Frontend | SPA con rutas protegidas (`PrivateRoute`, `Layout`, `AdminLayout`) y `@dnd-kit` para el Kanban |
| IA | Google Gemini: `gemini-3.8-flash`, con fallback a `gemini-3.6-flash` |
| Librerías | `tenacity` (reintentos), `pypdf` (lectura de PDF), ReportLab (DFR) |

Los documentos no nombran el framework de la API ni el del frontend. Django REST Framework y React son lo que sugieren `@dnd-kit` y `PrivateRoute`, pero confírmalo antes de crear el scaffolding.

## Reglas de negocio que no se negocian

### Aislamiento y roles

- Todo recurso se filtra por propietario: `proyecto.usuario == request.user`.
- El acceso a un recurso ajeno responde `404`, nunca `403`, para no revelar que existe (CA-05). Esto incluye el DFR.
- La separación de roles se aplica en el backend; la redirección de `PrivateRoute` es solo experiencia de usuario.
  - Usuario normal (`is_staff = False`): flujo de proyectos. Sin acceso a `/admin/*`.
  - Administrador (`is_staff = True`): usuarios y bitácora. No participa en el flujo de proyectos.
- Las rutas `/admin/*` son del frontend. No actives el admin nativo de Django.

### Autenticación

- Login por `username` o `email`. Access token de 30 minutos y refresh de 1 día, con rotación.
- 5 intentos fallidos por combinación de IP y usuario bloquean 15 minutos: responde `429` con `retry_after_seconds` y registra el evento en la bitácora.
- Las cuentas creadas por el administrador deben cambiar la contraseña temporal en su primer inicio de sesión. El autorregistro no pasa por este paso.
- OTP de recuperación con vigencia de 15 minutos.
- Contraseña: mínimo 8 caracteres, con mayúscula, minúscula, número y carácter especial, sin similitud a datos personales.

### Archivos

- Extensiones permitidas: `.txt`, `.pdf`, `.docx`, `.mp3`, `.wav`, `.m4a`. Máximo 25 MB. Valida ambas cosas en el backend.
- Ruta de almacenamiento: `documentos/{user_id}/{proy_id}/`.
- Al borrar un `Archivo` de la base de datos, borra también el archivo físico.
- Si un PDF no tiene capa de texto, informa que no contiene texto legible. No hay OCR.

### Extracción con IA

- `tipo` solo puede ser `Funcional` o `No Funcional`.
- `dificultad` solo puede ser `1, 2, 3, 5, 8, 13`. Cualquier otro valor se normaliza al válido más cercano; si equidista de dos, usa el mayor (`4` pasa a `5`).
- Reintenta 5 veces con backoff exponencial sobre el modelo principal ante `429` o `503`; si sigue fallando, cambia al modelo de respaldo.
- La extracción es síncrona. La meta de 30 segundos aplica sin reintentos; si se supera el tope máximo configurable, responde `504`.
- Envía las descripciones existentes del proyecto como contexto (`PROMPT_EXTRACCION_CON_CONTEXTO`) para deduplicar semánticamente y guardar solo requerimientos nuevos.
- Trata el contenido subido como no confiable: encapsúlalo entre delimitadores `<<<TEXTO_NO_CONFIABLE>>>`, indica en el prompt del sistema que se ignoren las órdenes dentro del documento y valida el JSON devuelto antes de guardarlo.

### Sprints

- Duración entre 7 y 14 días, validada en `Sprint.clean()`. Los generados automáticamente duran 7.
- Estados de requerimiento: `Disponible`, `En curso`, `Hecho`. Solo se pueden cambiar en el sprint con `is_active = True`.
- Subir, eliminar o reprocesar un archivo elimina los sprints del proyecto dentro de una transacción atómica (CA-03). Los requerimientos que permanecen quedan sin sprint y conservan su estado.
- Reprocesar reemplaza los requerimientos de ese archivo por los de la nueva extracción.
- Si la extracción falla, no se modifica ningún dato. Llama a Gemini fuera de la transacción.
- Crear un sprint manual cierra el activo y migra al nuevo lo que no esté en `Hecho`. Solo puede existir un sprint activo.
- `sincronizar_cronologia` se ejecuta bajo demanda al consultar el proyecto o sus sprints. Cierra el sprint vencido, crea los sprints de 7 días necesarios hasta la fecha actual y migra al nuevo sprint activo lo que no esté en `Hecho`.

### Cifrado y auditoría

- `POST /api/proyectos/secure/` cifra petición y respuesta con AES-256-GCM. Un payload sin cifrar responde `400`.
- La clave de 32 bytes es por sesión: la genera el backend al iniciar sesión y la entrega por HTTPS. El frontend la guarda solo en memoria; nunca va en el código ni en `localStorage`.
- Las señales `pre_save`, `post_save` y `post_delete` registran en `BitacoraMovimiento` todo cambio en `Proyecto`, `Archivo`, `Sprint`, `Requerimiento`, `User` y `PerfilUsuario`, con usuario, IP y valores anteriores y nuevos.
- `BitacoraMovimiento` no lleva llaves foráneas duras, para conservar el historial.
- No guardes contraseñas, hashes, tokens ni OTP en la bitácora, aunque `User` esté auditado.

## Fuera de alcance del MVP

No implementes lo siguiente, ni dejes preparada la estructura para ello, salvo que se pida:

- Colaboración multiusuario o proyectos compartidos.
- Edición manual de texto, tipo o Story Points de un requerimiento.
- Integraciones con Jira, Trello, GitHub, GitLab, Linear o Azure DevOps.
- Video, grabación en el navegador o reuniones en vivo.
- Notificaciones push o alertas por vencimiento de sprint.
- OCR de PDF escaneados.
- Cambios de estado en sprints cerrados.

## Decisiones pendientes

Confirma estos puntos antes de implementar lo que dependa de ellos:

- Framework de la API y del frontend (ver [Stack](#stack)).
- Valor del tope máximo de la extracción síncrona.
- Que `gemini-3.8-flash` y `gemini-3.6-flash` aceptan audio y salida JSON, con una llamada real antes de fijarlos en el código.

## Definición de terminado

- Cada cambio se verifica contra su criterio de aceptación del PRD.
- El aislamiento entre usuarios (CA-05) tiene pruebas automatizadas de seguridad.
- La llamada a Gemini se simula en las pruebas; no se usa la API real ni se consume cuota.

## Convenciones

- Documentación, mensajes al usuario y commits en español.
- Mantén en español los nombres de dominio del PRD (`Proyecto`, `Requerimiento`, `sincronizar_cronologia`), sin traducirlos ni renombrarlos.
- Commits con prefijo tipo Conventional Commits, como en el historial: `chore: se agregan documentos .md del DREA`.
- Credenciales fuera del repositorio: la API key de Gemini, la clave AES, el secreto JWT y los datos de correo van en variables de entorno.
- No subas al repositorio los archivos de `documentos/`: contienen datos de los usuarios.
