# MDAgenda

Agenda del director con solapas: **Tareas**, **Calendario de provincia**, **Feriados**, **Bitácora**, **Equipo**, **Docentes**, **Disponible**, **Licencias**, **Paro**, **Progreso**, **Estudiantes**, **Memoria Anual** y **Contactos**.

La aplicación funciona **100% local**: no depende de ningún servidor ni de Firebase. Los datos se guardan siempre en el navegador (localStorage) y, opcionalmente, también en un archivo dentro de una carpeta del disco que vos elijas (ver más abajo), como respaldo fuera del navegador.

---

## Guardado local y respaldo en carpeta

- Todo lo que cargás se guarda automáticamente en el navegador apenas lo tocás — no hace falta ningún botón de "Guardar" aparte.
- Arriba a la derecha del encabezado hay un botón **"📁 Conectar carpeta local"**. Al usarlo (solo funciona en Chrome, Edge o Brave), el navegador te pide elegir una carpeta del disco; a partir de ahí, cada vez que se guarda algo también se escribe una copia en un archivo `mdagenda_datos.json` dentro de esa carpeta. La próxima vez que abras la página, si el navegador todavía recuerda el permiso, se reconecta sola; si no, aparece un botón para confirmarlo con un clic.
- Si en algún momento necesitás pasar los datos a otra computadora, copiá ese archivo `mdagenda_datos.json` — es un respaldo completo y legible.
- Los distintos listados (estudiantes, escuelas de origen, cronograma de tareas, feriados, fechas de provincia) también se pueden exportar/importar en Excel desde sus propios botones, como respaldo adicional o para carga masiva.

## Publicar en GitHub Pages (opcional)

Como ya no depende de Firebase, publicar el sitio es más simple: Settings del repositorio → Pages → Source: Deploy from a branch → rama `main` → carpeta `/ (root)`. La dirección queda `https://tuusuario.github.io/turepo/`.

Importante: al ser 100% local, cada navegador/dispositivo donde se abra la página tiene **su propia copia de los datos**, sin sincronizarse entre sí. Si usás la app desde más de un dispositivo, usá la carpeta conectada (o el Excel de respaldo) para pasar los datos de uno a otro.

---

## Guía rápida de uso

- **Tareas**: alta con **+** (descripción, fecha límite, prioridad en estrellas 0–4, tipo de tiempo directivo, y opcionalmente para quién es: Escuela 370 o Supervisión — estas últimas se resaltan en celeste). Vencidas en rojo. Tocar la tarea para editarla. Marcar como lista manda el registro a la Bitácora. Ordenable por carga, prioridad o fecha. Filtros por destino arriba de la lista. Se puede importar un cronograma completo desde Excel.
- **Calendario de provincia** y **Feriados**: alta manual o importación desde Excel, con capas activables en el calendario.
- **Bitácora**: tareas terminadas agrupadas por semana y día. Filtro por día, edición y eliminación (a la papelera). El botón **"➕ Agregar registro"** permite cargar un ítem directamente con la fecha que quieras (no solo hoy). Selección de días → Word o copiar.
- **Equipo** y **Docentes**: ficha con contacto, dirección, zona (norte/sur/ambas), horario por día, funciones fijas con casillero, y tareas asignables con seguimiento propio.
- **Disponible**: arma sola la presencia semanal a partir de los horarios cargados.
- **Licencias**: alta por persona con artículo, descripción, certificado y pedido entregados. El artículo 36 se controla solo (máximo 2 por mes y 6 por año). Selección múltiple → Word o copiar.
- **Paro**: alta por persona y fecha. Listado exportable a Word o copiar.
- **Progreso**: objetivos institucionales con submetas, hilo de tiempo, % de cumplimiento. Exportable a Word.
- **Estudiantes**: ficha completa (datos personales, escuela de origen, modalidad, zona, contactos, certificado médico con aviso automático de vencimiento, CUD/PPI/Adecuación, POT, etc.), Lista de espera con orden automático y reordenamiento, y estudiantes en atención por zona. Importación/exportación por Excel en ambas vistas, con filtros de certificado y buscador en tiempo real.
- **Memoria Anual**: preguntas guía con respuestas editables, mapa con capas de estudiantes/escuelas, exportación a Word.
- **Contactos**: agenda propia con buscador en tiempo real.
- **⚙ Configuración**: tema claro/sepia/oscuro, cupos por zona, reinicio semanal de funciones, papelera de la bitácora.

---

## Notas

- Los archivos Word exportados son `.doc` (Word los abre sin problema; si avisa sobre el formato, aceptar).
- Este repositorio se administra con git: cambios nuevos se integran acá y se suben con `git add .`, `git commit -m "mensaje"`, `git push`.
