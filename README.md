# MDAgenda

Agenda del director con solapas: **Tareas**, **Bitácora**, **Equipo**, **Docentes**, **Disponible**, **Licencias**, **Paro** y **Progreso**. Los datos se sincronizan entre dispositivos usando Firebase (gratuito) y el sitio se publica en GitHub Pages.

> ⚠ Si `firebaseConfig` tuviera valores `"PEGAR_..."`, el sitio funciona en **modo local** (guarda solo en ese dispositivo). Actualmente ya está configurado con el proyecto `mdagenda-0802`.

---

## Roles y accesos

- **Administrador**: la cuenta `caseres.marcosdavid@gmail.com` ve y edita todo, incluida ⚙ Configuración.
- **Invitado**: cualquier otra cuenta de Google entra viendo solo Tareas, Bitácora, Equipo, Docentes y Disponible, y trabaja sobre **sus propios datos** (nunca ve los del administrador).
- **Colaborador**: desde ⚙ Configuración → Accesos, el administrador puede habilitarle a un correo puntual más pestañas (Licencias, Paro, Progreso), con permiso de edición en ellas.

## Configurar Firebase (si hay que rehacerlo desde cero)

1. https://console.firebase.google.com → crear proyecto → registrar app Web (sin Hosting) → copiar los 6 valores de `firebaseConfig` y pegarlos en `index.html`.
2. **Authentication → Comenzar → método Google → Habilitar → Guardar.**
3. **Firestore Database → Crear base de datos → modo producción.** En la pestaña **Reglas**, pegar:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /usuarios/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
    match /permisos/{correo} {
      allow read: if request.auth != null &&
        (request.auth.token.email == correo ||
         request.auth.token.email == "caseres.marcosdavid@gmail.com");
      allow write: if request.auth != null &&
        request.auth.token.email == "caseres.marcosdavid@gmail.com";
    }
  }
}
```

4. **Authentication → Configuración → Dominios autorizados**: agregar el dominio de GitHub Pages (`tuusuario.github.io`).

## Publicar en GitHub Pages

Settings del repositorio → Pages → Source: Deploy from a branch → rama `main` → carpeta `/ (root)`. La dirección queda `https://tuusuario.github.io/turepo/`.

---

## Guía rápida de uso

- **Tareas**: alta con **+** (descripción, fecha límite, prioridad en estrellas 0–4). Vencidas en rojo. Editar / eliminar / marcar como lista (con comentario, va a la Bitácora). Ordenable por carga, prioridad o fecha. Calendario mensual abajo con cantidad de tareas por día.
- **Bitácora**: tareas terminadas agrupadas por semana y día, con quién las hizo. Filtro por día, edición y eliminación (envía a la papelera). Selección de días → Word o copiar.
- **Equipo**: 2 POTs y 2 coordinadores. Ficha con contacto, dirección, zona (norte/sur, con color), horario de dos turnos por día, funciones fijas con casillero, y tareas asignables con seguimiento propio.
- **Docentes**: igual que Equipo, pero se agregan/eliminan libremente. Horario en horas cátedra (40 min) con materia principal y secundarias.
- **Disponible**: arma sola la presencia semanal a partir de los horarios. Semana navegable; por día, vista mensual y grilla horaria. Marca licencias y paros.
- **Licencias**: alta por persona con artículo (número + letra opcional), descripción, certificado y pedido entregados. Vista semanal y mensual. Selección múltiple → Word o copiar. El **artículo 36 se controla solo**: máximo 2 por mes y 6 por año por persona; si se excede, avisa y no deja cargar.
- **Paro**: alta por persona y fecha. Listado por día exportable a Word o copiar.
- **Progreso**: metas con submetas anidadas (fecha y número opcionales). Hilo de tiempo con marca de HOY, % de submetas cumplidas y % de tiempo transcurrido. Comentario opcional al marcar una submeta como cumplida. Exportar a Word o copiar.
- **⚙ Configuración** (solo administrador): tema claro/sepia/oscuro, reinicio semanal de funciones, papelera de la bitácora (restaurar o vaciar), y gestión de accesos de colaboradores.

---

## Notas

- Los archivos exportados son `.doc` (Word los abre sin problema; si avisa sobre el formato, aceptar).
- El plan gratuito de Firebase alcanza de sobra para este uso.
- Este repositorio se administra con git: cambios nuevos se integran acá y se suben con `git add .`, `git commit -m "mensaje"`, `git push`.
