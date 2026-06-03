# What to avoid: too much text with no spaces in between, no separation lines, huge ASCII unreadable table

Tengo el cuadro completo. Vamos a las fases de orientación.

Fase 1 — Mapa del feature
¿Qué hace "Correr Inferencia"?

Le permite al equipo NX tomar un clip de video grabado por el pipeline (MP4 en el Jetson) y volver a correrle todos los modelos — PeopleNet, edad/género, fall detection, re-ID — con los overlays visuales activados, para verificar que las detecciones son correctas sin tener que estar físicamente en el lugar del cliente ni esperar que vuelva a ocurrir el evento.

Componentes involucrados:

Componente	Rol en el flujo
streamlit_app.py	Dispara la solicitud y muestra el resultado
Redis	Canal de señalización entre contenedores
app.py	Pipeline live que debe apagarse para ceder el paso
docker-entrypoint.sh	Loop que coordina el cambio entre modos
app_video_testing.py	El pipeline de playback que corre la inferencia
El flujo de datos, paso a paso:


[Usuario: botón "▶ Correr Inferencia"]
         │
         ▼
Streamlit escribe Redis: nx:qa:playback_video = "/nx_tech/recordings/.../tiled.mp4"
         │
         ▼
app.py (live) tiene un thread "playback-watcher" que lee Redis cada 3 s
         │ detecta la key → llama GLib.idle_add(loop.quit)
         ▼
GStreamer main loop se detiene → app.py hace cleanup (workers, pipeline) → sys.exit(42)
         │
         ▼
docker-entrypoint.sh ve exit code 42 → no lo trata como error
         │ lee nx:qa:playback_video de Redis
         │ lee nx:qa:status para saber capabilities y client
         ▼
Lanza: app_video_testing.py --input <path> --capabilities ... --client ...
         │
         │ primero: pipeline.set_state(PAUSED) → TRT engines se compilan (puede ser lento)
         │ cuando PAUSED confirmado: escribe nx:qa:playback_info en Redis
         ▼
app_video_testing.py pone pipeline en PLAYING → inferencia corre sobre el MP4
         │ Probe A y Probe B funcionan igual que en modo live
         │ MJPEG server sirve el video con overlays en :8080
         ▼
Streamlit ve nx:qa:playback_info seteado → muestra el iframe MJPEG + log de detecciones
         │
         │ [video termina: EOS]
         ▼
app_video_testing.py sale → entrypoint borra nx:qa:playback_video de Redis
         │
         ▼
Siguiente iteración del while loop → no hay playback_video → arranca app.py de nuevo (live)
Punto de entrada: El usuario hace clic en streamlit_app.py:669 — ahí está el _r.set("nx:qa:playback_video", ...).

Blast radius: Cuando se activa, el pipeline live se apaga por completo durante la inferencia. El cliente en producción deja de recibir analytics mientras dura el playback. Por eso la función existe solo en QA mode.

Fase 2 — Las abstracciones clave que necesitas entender
1. Redis como señal de control entre contenedores

Redis no es solo cache de datos aquí — actúa como el canal de IPC entre el contenedor qa_app (Streamlit) y el contenedor deepstream. No hay ninguna llamada HTTP directa entre ellos: todo pasa por keys de Redis. La key nx:qa:playback_video es esencialmente una señal de semáforo: presente = "cambia a playback", ausente = "modo live".

La gotcha importante: si Redis cae mientras el pipeline está en playback, el entrypoint no puede borrar la key al terminar y el loop va a intentar arrancar playback de nuevo con el mismo video. El código tiene 2>/dev/null y || true en todos los lugares que tocan Redis para evitar que un fallo de Redis mate el loop.

2. El exit code 42 como protocolo

app.py usa sys.exit(42) como una señal privada hacia docker-entrypoint.sh. El entrypoint tiene set -e activo (sale ante cualquier error), por eso necesita set +e antes de ejecutar app.py. El código 42 no es un error — es "oye, hay trabajo de playback pendiente, no me reinicies solo, verificá Redis primero".

Podés ver exactamente ese patrón en docker-entrypoint.sh:144–153.

3. El pre-roll antes de PLAYING

app_video_testing.py hace algo no obvio: en lugar de ir directo a PLAYING, primero va a PAUSED y espera hasta 15 minutos. ¿Por qué?

Con filesrc, cuando el pipeline arranca en PLAYING, GStreamer empieza a leer el archivo inmediatamente. Si nvinfer todavía está compilando los TRT engines (primera corrida, o engines stale), el video llega a EOS antes de que la inferencia arranque — el archivo termina antes de que los modelos estén listos.

Al ir a PAUSED primero, GStreamer hace el "pre-roll": carga el primer frame y se queda esperando. Durante ese tiempo, nvinfer compila sus engines. Cuando get_state() confirma PAUSED, los engines ya están listos. Solo ahí se pasa a PLAYING.

El momento en que se escribe nx:qa:playback_info en Redis es deliberado: pasa después del pre-roll (app_video_testing.py:432). Streamlit muestra una pantalla de "Iniciando inferencia..." hasta que esa key aparece, para que el usuario no vea el feed live del RTSP y piense que algo está mal.

4. GLib.idle_add vs. llamar loop.quit directo

El playback-watcher corre en un thread Python separado. El GStreamer main loop corre en otro thread (el principal). Llamar loop.quit() desde un thread externo no es thread-safe en GLib. GLib.idle_add(loop.quit) encola la llamada para que el loop principal la ejecute en su próxima iteración disponible — es el patrón estándar de GLib para comunicarse con el main loop desde threads externos.

Fase 3 — Guía de lectura recomendada
Para entender este feature completamente, te recomiendo leer en este orden:

streamlit_app.py:656–678 — el botón "▶ Correr Inferencia" y lo que escribe en Redis. Es el punto de entrada del usuario.

streamlit_app.py:783–858 — cómo Streamlit decide qué mostrar según el estado de las dos keys (playback_video y playback_info). Tres estados: sin playback → lista de clips; transicionando → pantalla de espera; activo → iframe MJPEG.

app.py:652–705 — el playback-watcher y el exit 42. Notar el GLib.idle_add.

docker-entrypoint.sh:84–155 — el loop completo. Fijate cómo lee capabilities y client de nx:qa:status para que el playback use la misma config que el pipeline live.

app_video_testing.py:408–458 — el pre-roll y la escritura de playback_info. Este es el "handshake" que le dice a Streamlit que ya puede mostrar el video.

¿Hay alguna parte específica que quieras profundizar? ¿La lógica de estados en Streamlit, el loop del entrypoint, o cómo app_video_testing.py construye el pipeline de una forma distinta a app.py?
