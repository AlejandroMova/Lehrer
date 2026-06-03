# What to avoid: unreadable ASCII tables

Tengo todo lo que necesito. Te mapeo el flujo completo.

Phase 1 — Feature Map: "Correr Inferencia"
El objetivo de esta feature es: tomar un clip MP4 grabado en QA y pasarlo por el mismo pipeline de modelos que usa el stream en vivo, de modo que puedas ver las detecciones con overlays y los API calls que habría generado ese momento.

Los 5 componentes involucrados

Streamlit (qa_app)          Redis              docker-entrypoint.sh       app.py            app_video_testing.py
     │                        │                        │                     │                       │
     │ click "Correr"         │                        │                     │                       │
     │─── SET playback_video ─▶│                        │                     │                       │
     │                        │◀── poll cada 3s ────────────────────────────▶│                       │
     │                        │    (playback_video?)    │                     │                       │
     │                        │───────────── "sí" ──────────────────────────▶│                       │
     │                        │                        │      GLib.idle_add(quit)                    │
     │                        │                        │      sys.exit(42)   │                       │
     │                        │                        │◀─── loop detecta 42 │                       │
     │                        │◀── GET playback_video ─│                     │                       │
     │                        │◀── GET nx:qa:status ───│                     │                       │
     │                        │                        │─── python app_video_testing.py --input ... ─▶│
     │                        │                        │                                              │
     │◀── GET playback_info ──│◀────────── SET playback_info ───────────────────────────────────────▶│
     │  (muestra MJPEG)        │                        │                                              │
     │                        │                        │                     │      video termina      │
     │                        │                        │◀──────── app_video_testing.py termina ───────│
     │                        │◀── DEL playback_video ─│                     │                       │
     │                        │                        │── python app.py (live de nuevo) ────────────▶│
