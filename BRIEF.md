# Brief del Proyecto Integrador — DREA

> **Actividad A1 · Taller "Ya que vamos a pecar, hagámoslo bien"**  
> **Herramienta / Modelo recomendado:** Notion AI / Claude Sonnet o Gemini Pro (Modelo equilibrado/pesado para síntesis y estructuración de alcance).

---

## Tu Brief en 5 puntos

### 1. ¿Qué es, en una frase?
**DREA (Documentación de Requerimientos y Extracción Automatizada)** es una plataforma web que extrae automáticamente requerimientos de software (funcionales y no funcionales) a partir de documentos y grabaciones de audio usando Inteligencia Artificial (Google Gemini), los organiza en un tablero ágil de sprints y genera el Documento Formal de Requerimientos (DFR) en PDF.

### 2. ¿Para quién es y qué problema resuelve?
- **Para quién:** Analistas de sistemas, líderes de proyecto (Product Owners / Project Managers), desarrolladores y equipos estudiantiles o profesionales de ingeniería de software.
- **Qué problema resuelve:** Levantar, clasificar, estimar y documentar requerimientos a partir de minutas, entrevistas grabadas o documentos extensos es lento, propenso a omisiones y genera requerimientos duplicados o ambiguos. DREA automatiza la transcripción, extracción, deduplicación semántica y estimación en Story Points en segundos.

### 3. ¿Qué SÍ hace la primera versión (MVP)?
- **Autenticación y roles:** Registro e inicio de sesión con JWT, bloqueo temporal tras 5 intentos fallidos, recuperación de contraseña por correo (OTP) y separación estricta entre **Usuario Normal** y **Administrador**.
- **Gestión de proyectos (aislados por usuario):** Creación (con cifrado de payload AES-256-GCM), edición, consulta de métricas en Dashboard y eliminación de proyectos.
- **Extracción con IA multimodal y deduplicación:** Carga de archivos de texto (`.txt`, `.pdf`, `.docx`) y audio (`.mp3`, `.wav`, `.m4a`) de hasta 25 MB; extracción con Gemini (`gemini-3.8-flash` con fallback a `gemini-3.7-flash`), clasificación en *Funcional* / *No Funcional*, asignación de dificultad en escala Fibonacci (`1, 2, 3, 5, 8, 13` Story Points) y **deduplicación semántica** contra requerimientos ya existentes en el proyecto.
- **Gestión de Sprints y Tablero Kanban:** Generación de tablero inicial, creación de sprints (de 7 a 14 días), sincronización cronológica automática (migrando requerimientos pendientes al nuevo sprint activo cuando vence el anterior) y cambio de estado drag-and-drop (*Disponible*, *En curso*, *Hecho*).
- **Reporte DFR y Auditoría:** Descarga de reporte ejecutivo DFR en formato PDF y panel de administración con CRUD de usuarios y bitácora automática (`INSERT`, `UPDATE`, `DELETE`).

### 4. ¿Qué NO hace (todavía)?
- **No permite trabajo colaborativo multiusuario en un mismo proyecto** (cada proyecto pertenece exclusivamente al usuario que lo creó; no hay asignación de tareas a distintos miembros del equipo).
- **No edita manualmente el texto ni los Story Points individuales de un requerimiento** una vez extraído por la IA (solo permite cambiar su estado en el sprint activo o reprocesar el archivo).
- **No se integra con herramientas externas de gestión** como Jira, Trello, GitHub Issues o Azure DevOps.
- **No procesa grabaciones de video ni reuniones en vivo en tiempo real** (únicamente archivos estáticos subidos a la plataforma).
- **No permite modificar el estado de requerimientos que pertenezcan a sprints cerrados/inactivos.**

### 5. ¿Cómo sabremos que funciona?
- Un usuario puede registrarse, crear un proyecto y subir un documento (`.pdf`/`.docx`/`.txt`) o audio (`.mp3`/`.wav`/`.m4a`) menor a 25 MB, obteniendo en menos de 30 segundos una lista estructurada de requerimientos con tipo y Story Points válidos sin duplicados semánticos.
- El usuario puede generar el tablero Kanban, mover un requerimiento del sprint activo de *Disponible* → *En curso* → *Hecho*, y descargar un archivo PDF (DFR) válido y legible con las métricas y tablas actualizadas.
- Un usuario no puede acceder ni modificar proyectos de otro usuario (validado con pruebas automatizadas de seguridad), y el Administrador puede auditar cada movimiento en la bitácora.

---

## Las 3 preguntas de ajuste del Brief (Paso 4 de la Actividad A1)

1. **¿Qué ocurre con el tablero de sprints si el usuario sube un archivo nuevo o elimina uno existente cuando el proyecto ya había avanzado?**  
   *Respuesta incorporada:* Por diseño del MVP, subir, eliminar o reprocesar un archivo reinicia los sprints del proyecto para recalcular y mantener la consistencia del backlog antes de volver a generar el tablero.
2. **¿Qué pasa si la API de Gemini falla por límite de cuota (error 429) o caída temporal del servicio (503)?**  
   *Respuesta incorporada:* El sistema reintenta hasta 5 veces con *backoff* exponencial usando `gemini-3.8-flash` y, si persiste el fallo, cambia automáticamente al modelo de respaldo `gemini-3.7-flash` sin perder la transacción.
3. **¿El administrador puede crear proyectos y subir archivos igual que un usuario normal?**  
   *Respuesta incorporada:* No. Los roles están estrictamente separados: el Administrador es redirigido exclusivamente a `/admin/dashboard` (gestión de usuarios y bitácora de auditoría), mientras que el flujo de proyectos es exclusivo para usuarios estándar.

