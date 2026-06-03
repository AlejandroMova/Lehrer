# What to do: clear text, spaced correctly with lines in between topics. Text is concise but useful

Tengo suficiente contexto para explicarte el flujo completo. Aquí va.

Phase 1 — Feature Map: "Correr Inferencia"
Esta feature tiene una sola responsabilidad: volver a correr los modelos de IA sobre un clip de video grabado, en vez de sobre las cámaras en vivo. Sirve para verificar que el pipeline está detectando bien o para depurar casos edge.

El flujo completo atraviesa 5 componentes en secuencia:

1. El botón (Streamlit — streamlit_app.py:666–672)
Cuando el técnico hace click en ▶ Correr Inferencia, Streamlit ejecuta exactamente esto:


_r.set("nx:qa:playback_video", video_options[selected_idx][1])
Un SET en Redis con la key nx:qa:playback_video y como valor la ruta absoluta del archivo MP4 dentro del contenedor (ej: /nx_tech/recordings/20260602_143000/tiled.mp4).

Eso es todo lo que hace Streamlit. No habla con el pipeline directamente. No envía un comando. Solo escribe una key en Redis y espera.

2. El watcher (thread en app.py — líneas 655–686)
Mientras el pipeline live está corriendo, hay un thread daemon llamado _playback_watcher que cada 3 segundos hace GET nx:qa:playback_video en Redis.

Cuando detecta que la key apareció:

Setea _exit_for_playback[0] = True
Llama GLib.idle_add(loop.quit) — que es la forma thread-safe de decirle al GLib main loop que se detenga
Nota clave: no puede llamar loop.quit() directo desde el thread — GStreamer es single-threaded en su event loop. idle_add encola la llamada para que el loop principal la ejecute él mismo en su próxima iteración.

3. El shutdown de app.py y el exit code 42 (líneas 692–705)
Cuando el GLib loop termina, el código entra al bloque finally y hace cleanup normal del pipeline:


finally:
    pipeline.set_state(Gst.State.NULL)
    api_client.stop()
    stop_workers()
    ...

if _exit_for_playback[0]:
    sys.exit(42)
El pipeline se detiene limpiamente. Luego app.py sale con código de salida 42. Ese número es una convención interna — es la señal que el entrypoint usa para distinguir "quiero cambiar a playback" de un error real.

4. El entrypoint detecta el exit 42 y arranca playback (docker-entrypoint.sh:139–154)
El entrypoint tiene el siguiente patrón:


set +e
"$@"          # esto es app.py corriendo
EXIT_CODE=$?
set -e

if [ $EXIT_CODE -ne 0 ] && [ $EXIT_CODE -ne 42 ]; then
    exit $EXIT_CODE   # error real → salir
fi
# si fue 42, el while true itera y...
En la próxima iteración del while true, el entrypoint hace GET nx:qa:playback_video en Redis. Como la key todavía está ahí (Streamlit la dejó y app.py no la borró), encuentra el path del video y lanza:


python3 /nx_tech/pipelines/app_video_testing.py \
    --input "$PLAYBACK_VIDEO" \
    --capabilities "$PLAYBACK_CAPS" \
    --client "$PLAYBACK_CLIENT" \
    --no-loop
Las capabilities y el cliente los lee de nx:qa:status en Redis — que app.py publicó al arrancar — así el pipeline de playback usa exactamente los mismos modelos que el pipeline live.

5. El pipeline de playback corre y termina (app_video_testing.py)
app_video_testing.py es una variante del pipeline live pero con filesrc + decodebin en vez de rtspsrc. Corre los mismos modelos (PeopleNet, age/gender, etc.) sobre el video, emite los mismos eventos al backend, y cuando el archivo termina, sale con código 0.

Después, el entrypoint borra la key:


redis-cli DEL nx:qa:playback_video
Y en la siguiente iteración del while true, como la key ya no existe, vuelve a lanzar app.py — el pipeline live se restaura.

El flujo completo en una línea por paso:

[Streamlit] SET nx:qa:playback_video → ruta del MP4
     ↓
[app.py watcher thread] detecta la key cada 3s → GLib.idle_add(loop.quit)
     ↓
[app.py main] GLib loop se detiene → cleanup → sys.exit(42)
     ↓
[entrypoint] detecta exit 42 → siguiente iteración → lee la key → lanza app_video_testing.py
     ↓
[app_video_testing.py] corre modelos sobre el clip → termina → exit 0
     ↓
[entrypoint] borra la key → siguiente iteración → lanza app.py (live restaurado)
Las partes que más confunden a primera vista
¿Por qué Redis en vez de una señal directa? Porque Streamlit y el pipeline DeepStream corren en contenedores Docker distintos — no comparten memoria, no comparten proceso. Redis es el bus de comunicación entre ellos.

¿Por qué exit code 42? El docker-entrypoint.sh tiene set -e activo, que mata el proceso si cualquier comando falla. Sin la distinción entre "error real" (exit ≠ 0 y ≠ 42) y "cambio de modo" (exit 42), el entrypoint moriría cuando app.py terminara voluntariamente.

¿Por qué GLib.idle_add en vez de loop.quit() directo? GStreamer no es thread-safe. El watcher es un thread Python separado. Llamar loop.quit() desde ese thread puede corromper el estado del GLib main loop. idle_add encola la llamada para que el loop la ejecute él mismo.

Vocabulary Table

Term | Library / Origin | What it is | How it's used in this code
-----|-----------------|------------|---------------------------
GLib.idle_add | GLib (PyGObject) | Encola una función para que el GLib main loop la ejecute en su próxima iteración | Usado en _playback_watcher para llamar loop.quit() de forma thread-safe desde el watcher thread
nx:qa:playback_video | [this codebase] | Key de Redis que actúa como trigger del modo playback | Streamlit la escribe con el path del MP4; el entrypoint la lee para saber qué video correr; se borra cuando playback termina
_exit_for_playback | [this codebase] | Lista de un elemento usada como flag mutable entre threads | Se setea a True en el watcher thread; se lee en el main thread después del GLib loop para decidir si llamar sys.exit(42)
sys.exit(42) | Python stdlib | Termina el proceso con código de salida 42 | Convención interna para señalar al entrypoint que el exit fue intencional, no un error
filesrc + decodebin | GStreamer | Elementos para leer y decodificar un archivo de video local | Usados en app_video_testing.py como reemplazo de rtspsrc para procesar MP4 en vez de stream RTSP en vivo
nx:qa:status | [this codebase] | Key de Redis donde app.py publica su configuración al arrancar | El entrypoint la lee al lanzar playback para pasar las mismas capabilities y cliente a app_video_testing.py
--no-loop | [this codebase] | Flag de app_video_testing.py | Hace que el pipeline termine cuando el video llega a EOS; necesario para que el entrypoint detecte que playback terminó
GLib.MainLoop | GLib (PyGObject) | Event loop que mantiene el pipeline GStreamer corriendo | Se llama .run() al final de main() — bloquea hasta que alguien llama .quit()


¿Hay alguna parte que quieras que expliquemos más a fondo — el pipeline de app_video_testing.py, cómo Streamlit muestra el estado durante el playback, o algo sobre la comunicación Redis?
