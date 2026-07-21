# MDAgenda — Documentación técnica (para lectura por una IA)

Este documento describe con precisión qué es MDAgenda, cómo está construida y cómo funciona cada parte, para que otra IA (o un desarrollador) pueda entender el sistema sin tener que leer todo el código fuente primero.

## 1. Qué es

MDAgenda es una agenda de gestión para un director de escuela domiciliaria y hospitalaria (dos sedes). Le permite administrar tareas propias y del equipo, un registro histórico de lo realizado, fichas de personal (POT, coordinadores, docentes), disponibilidad horaria semanal, licencias/ausencias con control normativo, paros, y seguimiento de metas de gestión a lo largo del tiempo.

Es una **aplicación web de una sola página** (`index.html`), sin build ni dependencias instaladas: HTML + CSS + JavaScript "vanilla" en un único archivo, con módulos ES cargados por CDN solo para Firebase. No hay backend propio: toda la persistencia va directo a Firebase (Firestore + Authentication) desde el navegador.

## 2. Arquitectura y despliegue

- **Archivo único**: `index.html` (~2600 líneas: ~640 de CSS embebido, el resto HTML + `<script type="module">`).
- **Hosting**: GitHub Pages, repositorio `Davidelcheff456/MDAgenda`, publicado en `https://davidelcheff456.github.io/MDAgenda/`.
- **Backend**: Firebase (proyecto `mdagenda-0802`), usado solo como servicio externo:
  - **Authentication**: login con Google (popup, con fallback a `signInWithRedirect` si el popup es bloqueado).
  - **Firestore**: dos colecciones.
    - `usuarios/{uid}`: un documento por usuario autenticado, contiene el `estado` completo (ver sección 3) serializado como JSON. Se sincroniza en tiempo real con `onSnapshot`.
    - `permisos/{correoEnMinusculas}`: documentos que listan qué pestañas puede usar un colaborador que no es el administrador (ver sección 6).
  - Reglas de Firestore: cada usuario solo puede leer/escribir su propio documento en `usuarios`; `permisos` solo lo escribe el administrador, pero cualquier usuario autenticado puede leer su propia entrada.
- **Modo local**: si `firebaseConfig.apiKey` empieza con `"PEGAR"` (placeholder sin completar), la app cae a modo local: usa `localStorage` (clave `"mdagenda"`) en vez de Firestore, y no hay login. Actualmente el proyecto tiene la config real cargada, así que corre en modo Firebase.
- **Guardado**: cualquier cambio de datos llama a `guardar()`, que si hay sesión Firebase activa hace un `setDoc` debounced (400 ms) al documento del usuario; si no, escribe a `localStorage`.

## 3. Modelo de datos (`estado`)

Un único objeto JS `estado` en memoria contiene todo. Se serializa entero a Firestore/localStorage. Forma (ver `estadoInicial()`):

```
estado = {
  tareas: [ Tarea ],       // tareas propias del director
  bitacora: [ EntradaBitacora ],
  equipo: [ Persona ],     // longitud variable, ver abajo
  docentes: [ Persona ],
  licencias: [ Licencia ],
  paros: [ Paro ],
  papelera: [ EntradaBitacora & {borradaEl} ],
  metas: [ Meta ],
  opciones: { tema: "claro"|"sepia"|"oscuro", resetFunciones: boolean }
}
```

### Tarea
```
{ id, texto, fecha: "YYYY-MM-DD"|null, estrellas: 0-4, creada: timestamp }
```
Puede pertenecer a un "owner": `"propia"` (tareas del director) o `"eq:<id>"` / `"doc:<id>"` (tareas asignadas a una persona de Equipo o Docentes; ver `personaDe()` / `listaDe()`). Las tareas de una persona viven dentro de `persona.tareas`, no en `estado.tareas`.

### EntradaBitacora
```
{ id, texto, comentario, fechaFin: "YYYY-MM-DD", estrellas, persona: "Nombre Apellido"|"", rol: "POT"|"Coordinador/a"|"Docente"|"Director"|"" }
```
Se crea al marcar una tarea (propia o de alguien) como lista, vía modal de comentario opcional. `persona`/`rol` quedan grabados como texto plano (snapshot), no como referencia.

### Persona (Equipo o Docentes)
```
{
  id, rol,                 // "Director" | "POT" | "Coordinador/a" | "Docente"
  nombre, apellido, telefono, correo, notas,
  direccion,                // texto libre, dónde vive
  zona: ""|"norte"|"sur",   // colorea el nombre en toda la app (azul marino / verde)
  horarios: {
    lun|mar|mie|jue|vie|sab|dom: {
      desde, hasta,          // HH:MM, turno 1 — POT/Coordinador/Director
      desde2, hasta2,        // HH:MM, turno 2 opcional (ej. tarde)
      // para Docentes en vez de hasta/hasta2:
      modulos, modulos2       // cantidad de horas cátedra (40 min cada una); la hora de salida se calcula
    }
  },
  tareas: [ Tarea ],
  funciones: [ Funcion ]     // tareas fijas/recurrentes con casillero
}
```
- `estado.equipo` siempre incluye un "Director" (id `dir1`) más los roles fijos POT×2 y Coordinador/a×2 (`equipoInicial()`); `migrarEquipo()` agrega el Director automáticamente si el dato guardado es viejo y no lo tiene.
- `estado.docentes` es una lista libre (agregar/quitar), y además tiene `materia` (string) y `materiasSec` (string, separadas por coma).
- La disponibilidad horaria (pestaña Disponible) se calcula 100% a partir de `horarios`, no se carga por separado.

### Funcion
```
{ id, texto, hecha: "" | "YYYY-MM-DD" (lunes de la semana en que se tildó) }
```
Tareas fijas que se repiten (ej. "Firmar planillas"). Se muestran con checkbox; al tildarla se tacha. Si `estado.opciones.resetFunciones` es true, las funciones de personas que **no** son Docente (`p.rol !== "Docente"`) se consideran "tildadas" solo si `hecha === lunesDeHoy` — es decir, se destildan visualmente cada lunes aunque el dato no se borre. Las funciones de Docentes nunca se auto-reinician.

### Licencia
```
{
  id, persona: "eq:<id>"|"doc:<id>",
  desde: "YYYY-MM-DD",
  hasta: "YYYY-MM-DD"|null,   // null si continua=true
  continua: boolean,          // sin fecha de fin todavía (licencia abierta)
  numero: string,             // ej. "36", "29" — número de artículo/licencia
  letra: string,               // ej. "C" — opcional
  descripcion: string,
  ausente: boolean,            // true = falta sin aviso (no es una licencia formal, pero se registra en la misma colección)
  cert: boolean,                // entregó certificado
  pedido: boolean               // entregó el pedido/formulario
}
```
`tipo` es un campo legado (versión anterior) que se usa como fallback de `descripcion` si existe (`descLic()`). `finLic(l)` devuelve `"9999-12-31"` si `continua` es true, para que la licencia se considere vigente indefinidamente en los cálculos de solapamiento.

**Regla de negocio del artículo 36** (`validar36()`): a una persona no se le pueden cargar más de **2 días de licencia número "36" por mes** ni más de **6 por año**. Se valida contando días individuales (`diasEntre(desde,hasta)`), no registros, sumando lo ya existente más los días de la licencia que se está por guardar. Si se excede, `guardarLicenciaModal()` cancela el guardado y muestra un `alert()` con el detalle (cuántos ya tiene y en qué mes/año).

### Paro
```
{ id, persona: "eq:<id>"|"doc:<id>", fecha: "YYYY-MM-DD" }
```
Un registro por persona y día (no se permite duplicar la misma combinación).

### Meta / Submeta (Progreso)
```
Meta = { id, titulo, submetas: [ Submeta ] }
Submeta = { id, texto, fecha: "YYYY-MM-DD"|"", numero: number|null, hecha: boolean, comentario: string }
```
El progreso de una meta es `submetasHechas / totalSubmetas` (%). Si hay al menos 2 submetas con fecha, se calcula además un "% de tiempo transcurrido" entre la fecha más temprana y la más tardía, comparado con hoy (barra separada, en azul). Al tildar una submeta se abre un modal para agregar un comentario opcional (igual mecánica que completar una tarea).

## 4. Las 8 pestañas (`nav.tabs`, `data-tab`)

Cada botón de pestaña tiene un color pastel propio vía variable CSS `--ct`. El panel visible se controla con la clase `.activa` en `section.panel`; el cambio de pestaña es JS puro (sin routing/URL).

1. **Tareas** (`tareas`): alta rápida (+, con fecha límite y prioridad en estrellas 0–4), lista ordenable por orden de carga / prioridad / fecha (con toggle de dirección), tarjeta editable/eliminable/completable, y un **calendario mensual** propio (no nativo) debajo que muestra insignias con la cantidad de tareas por día (rojo si hay vencidas) y al hacer clic en un día abre un modal con el detalle.
2. **Bitácora** (`bitacora`): historial de tareas completadas, agrupado por semana → día. Filtro por día puntual (`bitFiltro`). Cada entrada es editable (texto/comentario/fecha) o se puede enviar a la papelera. Selección de días con checkbox → exportar a Word (`.doc` vía Blob) o copiar al portapapeles.
3. **Equipo** (`equipo`): tarjetas de Director + POT×2 + Coordinador/a×2. Cada tarjeta abre una ficha modal con datos de contacto, dirección, zona, horario (2 turnos por día), funciones fijas y notas. Debajo de la ficha (o en la tarjeta) se le pueden asignar tareas a esa persona, con el mismo flujo de completar/editar que las tareas propias.
4. **Docentes** (`docentes`): igual que Equipo pero la lista es libre (agregar/eliminar), con materia principal/secundarias y horario en horas cátedra (bloques de 40 min) en vez de hora de salida directa.
5. **Disponible** (`disponible`): vista de "quién está en la escuela y cuándo", calculada 100% a partir de los horarios cargados en Equipo/Docentes. Navegación por semana (anterior/actual/siguiente) mostrando tarjetas por día; al entrar a un día se ve una grilla por franjas horarias (por defecto 8:00–18:00, ajustada a bloques de `DUR_MODULO`=40 min) con quién está presente en cada franja. Marca automáticamente a quien tiene licencia (tachado, ámbar) o paro (tachado, rojo, negrita) ese día.
6. **Licencias** (`licencias`): alta de licencia/ausencia por persona (fecha de inicio, checkbox "continúa" sin fin, checkbox "ausente" para faltas sin aviso, número + letra + descripción, certificado/pedido entregados). Vista semanal y vista mensual (con selección múltiple de registros → exportar a Word o copiar). Aplica automáticamente la regla del artículo 36 (ver sección 3).
7. **Paro** (`paro`): alta por persona + fecha (sin duplicados). Selector de un día puntual para ver el listado de quiénes pararon, exportable a Word/copiar. Tabla con todos los registros históricos.
8. **Progreso** (`progreso`): metas con submetas anidadas. Cada meta se muestra como una tarjeta con barra de % cumplido (y barra de % de tiempo si corresponde) y un timeline vertical de submetas ordenadas por fecha, con un marcador "HOY" insertado en su posición cronológica. Exportable completo a Word o portapapeles, agrupado por año.

Además, en el header hay un botón **⚙ Opciones** (visible solo para el administrador) que abre un modal con: selector de tema (claro/sepia/oscuro), checkbox de auto-reinicio semanal de funciones, la **papelera** de la bitácora (restaurar entrada por entrada o vaciar todo), y la gestión de **accesos de colaboradores** (ver sección 6).

## 5. Renderizado

No hay framework: todo el DOM de listas/tablas se reconstruye con `document.createElement` + `innerHTML` puntual dentro de funciones `render*()` (una por sección: `renderTareas`, `renderBitacora`, `renderEquipo`, `renderDocentes`, `renderDisponible`, `renderLicencias`, `renderParo`, `renderProgreso`, `renderPapelera`...). `renderTodo()` las llama a todas juntas y se invoca después de cualquier cambio de estado remoto (snapshot de Firestore) o de arranque. Los cambios locales (clicks) llaman a la función `render*` específica en vez de `renderTodo()` completo, salvo cuando el cambio afecta a varias pestañas a la vez (p. ej. guardar una ficha de persona).

## 6. Roles y permisos

- **Administrador**: la cuenta con email exactamente igual a `ADMIN_EMAIL` (`caseres.marcosdavid@gmail.com`, comparado en minúsculas). Ve y edita las 8 pestañas y el botón ⚙ Opciones. Es dueño del documento `usuarios/{suUid}` en Firestore, que es el que en la práctica contiene "los datos reales" de la escuela.
- **Invitado**: cualquier otra cuenta de Google que inicia sesión sin tener un documento en `permisos/{correo}`. Ve solo `PESTANAS_INVITADO` = Tareas, Bitácora, Equipo, Docentes, Disponible (sin Licencias/Paro/Progreso/Configuración). **Importante**: un invitado trabaja sobre su **propio** documento `usuarios/{suUid}` — es una instancia de datos completamente separada de la del administrador, vacía al principio. No hay dato compartido salvo que el administrador le dé acceso.
- **Colaborador**: un invitado al que el administrador le agregó un documento en `permisos/{correo}` con una lista de pestañas (`{pestanas: [...]}`) desde el modal de Opciones → Accesos. Sigue operando sobre su propio documento de usuario, pero puede ver/usar más pestañas que un invitado común (las que el admin le haya marcado). No hay un modo donde un colaborador vea los datos del administrador: cada cuenta de Google tiene su propio silo de datos en `usuarios/{uid}`.
- Sin sesión iniciada (Firebase activo pero no logueado): se muestran las 8 pestañas (modo "demo" local, sin el gear de Opciones) para que cualquiera pueda ver la interfaz antes de loguearse; los datos ahí son efímeros (no persisten entre visitas si no hay sesión ni Firebase local storage aplicado).

Esto significa que **el control de acceso es de interfaz, no de seguridad estricta de datos entre cuentas**: cada cuenta de Google ve y modifica únicamente su propio documento en Firestore; las reglas de Firestore son las que garantizan que nadie pueda leer o escribir el documento de otro `uid`.

## 7. Exportaciones a Word/portapapeles

Todas las exportaciones a "Word" generan un archivo `.doc` (no `.docx`) armando un string HTML con estilos inline y creando un `Blob` de tipo `application/msword`, descargado vía un `<a download>` sintético. No usa ninguna librería externa. Existen en: Bitácora (`descargarWord`, por días seleccionados), Licencias (`licenciasWord`, por registros seleccionados, agrupado por mes), Paro (`paroWord`, por día), Progreso (`progresoWord`, todo agrupado por año). Cada una tiene su par `*Copiar()` que arma texto plano y usa `navigator.clipboard.writeText`.

## 8. Detalles de UI relevantes

- Colores de zona: `.zona-norte` = azul marino (`#12275e`, casi negro), `.zona-sur` = verde (`#2e8b57`); se aplican como clase en cualquier lugar donde se muestra el nombre de una persona (tarjetas, Disponible, Licencias, Paro).
- Temas: clases `body.tema-sepia` / `body.tema-oscuro` sobrescriben variables CSS (`--papel`, `--borde`, etc.) y estilos puntuales; el tema por defecto es claro (sin clase).
- Fechas: se manejan como strings `"YYYY-MM-DD"` (formato ISO de `<input type="date">`) en todo el estado, y se formatean para mostrar con `Intl`/`toLocaleDateString("es-AR", ...)` vía `fmtCorto`, `fmtLargo`, `mesLargo`, `etiquetaSemana`.
- Los `<input type="date">` y `<input type="time">` son nativos del navegador (sin librería de calendario custom), salvo el calendario visual de la pestaña Tareas que sí es una grilla propia solo de lectura/insignias.
- Los `confirm()`/`alert()` nativos del navegador se usan para confirmaciones de borrado y para los avisos de límite del artículo 36 o errores de login.

## 9. Puntos a tener en cuenta si se sigue desarrollando

- El archivo es monolítico (un solo `<script type="module">`); cualquier cambio debe mantenerse dentro de ese bloque salvo que se decida modularizar.
- No hay build ni linter configurado; los cambios se prueban abriendo el HTML (local o publicado en GitHub Pages) directamente.
- El login con Google **no funciona** abriendo el archivo como `file://` local (restricción de OAuth de Google) ni en dominios no agregados a Firebase Authentication → Dominios autorizados.
- Todo el estado vive en un único documento de Firestore por usuario; no hay paginación ni límites de tamaño manejados explícitamente — si la cantidad de datos creciera mucho (años de bitácora, por ejemplo), convendría revisar el límite de 1 MB por documento de Firestore.
- El repositorio local de referencia es `D:\APU\Agenda\MDAgenda` (clonado de `github.com/Davidelcheff456/MDAgenda`), que es la base de trabajo vigente.
