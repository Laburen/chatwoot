# Parches de Laburen sobre Chatwoot (fork 4.6.0)

Cambios de código que tenemos encima de Chatwoot y que no vienen en esta versión de upstream (o todavía no están
mergeados ahí). Leer esto antes de cada upgrade.

## "Escribiendo…" en WhatsApp

Para qué sirve: la app de Laburen llama `toggle_typing_status` con `notify_provider: true` usando el token del agent
bot. Chatwoot busca el último mensaje entrante de la conversación (el wamid en `source_id`) y le pide a Meta "leído +
escribiendo". El cliente ve "escribiendo…" en WhatsApp mientras el agente de IA arma la respuesta. El panel de
Chatwoot nunca manda el flag, así que lo que tipean las personas no llega a Meta.

Se arma con dos PRs de upstream y una adaptación nuestra:

### 1. PR #13705 — los bots pueden llamar `toggle_typing_status`

- https://github.com/chatwoot/chatwoot/pull/13705 (mergeado en upstream, viene desde 4.11.2). Commit `397b0bcc9d`.
- Sin esto, un token de bot recibe 401 en ese endpoint, y si se habilitara solo el permiso, el listener del panel
  rompe (500) al buscar el `pubsub_token` del bot.
- Código: `access_token_auth_helper.rb`, `conversations_controller.rb`, `action_cable_listener.rb`,
  `typing_status_manager.rb`. Trae specs y swagger (y unas líneas en `AGENTS.md`).

### 2. PR #16072 — `notify_provider` le avisa a Meta

- https://github.com/chatwoot/chatwoot/pull/16072 (abierto, sin review de los mantenedores).
- Commits traídos con cherry-pick (sin el merge de develop `eda90492d`):
  - `abc70754eb` — `WhatsappCloudService#send_typing_indicator` (Graph API v25.0, timeout 5 s, nunca lanza).
  - `539f9af93b` — `Conversations::TypingStatusManager` reenvía el "escribiendo" a Meta cuando viene `notify_provider`.
- Conflictos al traerlo: en `whatsapp_cloud_service.rb` y su spec el PR mete su método al lado de los métodos CSAT
  (`create_csat_template`, `delete_csat_template`, `get_template_status`), que acá no existen. Se dejó todo lo de la
  fork (por ejemplo `media_url(media_id, phone_number_id = nil)`) y se sumó **solo** `send_typing_indicator` y sus tests.

### 3. Adaptación nuestra — `phone_id_path(version = 'v13.0')`

- El PR llama `phone_id_path('v25.0')`, pero en 4.6.0 el método no aceptaba parámetros (el `rescue` del PR lo tapaba
  y el aviso nunca salía). Se pasó a la misma firma que upstream. Los que lo llaman sin parámetro siguen en v13.0.

## Checklist para cada upgrade de Chatwoot

1. ¿Upstream mergeó #16072? Si sí y es igual a lo nuestro → borrar nuestro parche. Si cambió el nombre del parámetro
   → ajustar la app de Laburen (si no, el aviso deja de salir **sin error**).
2. Si no lo mergeó, re-aplicar los 2 commits del PR sobre la versión nueva.
3. Si el upgrade es a ≥ 4.15, #13705 y la firma nueva de `phone_id_path` ya vienen (#13705 desde 4.11.2,
   `phone_id_path(version)` desde 4.14.2): quedarse con la versión de upstream y descartar las nuestras.
4. Conflictos esperables: `typing_status_manager.rb` casi no cambia upstream; `whatsapp_cloud_service.rb` cambia
   seguido, pero el PR solo agrega un método y toca la línea 1 (comentario de rubocop) → 1–2 bloques, se resuelve en
   minutos.
