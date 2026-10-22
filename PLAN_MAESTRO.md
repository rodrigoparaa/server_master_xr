# Plan maestro: Aegis Hardware Enclave

Guía viva para continuar el proyecto. Recoge **qué existe**, **qué principios no se negocian**, **qué falta** y, en detalle, **cómo construir el servidor de archivos**. Cada fase tiene tareas, criterios de terminado y pruebas. Actualiza las casillas y la sección «Estado» al cerrar trabajo.

---

## 1. Estado actual (22/10/2026)

| Área | Estado |
|---|---|
| Bóveda de notas cifradas (ECIES P-256 + ChaCha20-Poly1305, Secure Enclave) | ✅ En producción |
| RAM sin texto en claro en disco, TTL 15 min, purga por acelerómetro | ✅ |
| Graph-RAG híbrido (BM25 + grafo) con limpieza de fragmentos | ✅ |
| IA local Qwen2.5-1.5B en CPU, pesos en mmap (~160–190 MiB de huella) | ✅ 6–11 s por respuesta |
| Token por dispositivo (Llavero) y emparejamiento por código de 6 dígitos | ✅ |
| HTTPS con certificado propio fijado en el Mac | ✅ |
| Órdenes delicadas firmadas por el Secure Enclave del Mac (Touch ID) | ✅ |
| Protección de memoria (margen real, 507, descarga del modelo) | ✅ (avisos de iOS sin probar en real) |
| App iPhone «papel de seguridad», barra de menús macOS, centinela | ✅ |
| Repositorio único (`iOSApp/` incluido), licencia MIT | ✅ |

## 2. Principios que no se negocian

1. **El Secure Enclave del XR es la llave.** Nada que deba ser secreto se guarda en claro en disco ni sale del XR sin cifrar.
2. **Cero secretos en el código ni en git.** Tokens, Team ID, IP y huellas viven en el Llavero o en `~/.config/aegis/config.json` (600).
3. **Toda orden destructiva o que abre datos se firma** con la clave del Secure Enclave del dispositivo cliente (Touch ID / Face ID), con hora ±2 min y nonce de un solo uso.
4. **Transporte: HTTPS con certificado fijado + Tailscale.** Nunca exponer el 8443 a internet.
5. **La IA no se ralentiza por otras funciones.** Nada de trabajo pesado continuo en el XR (miniaturas, OCR, etc. se hacen en el cliente).
6. **Medir antes de afirmar.** Cada cambio de rendimiento o calidad se valida con datos en el iPhone real.
7. **Interfaz:** sistema «papel de seguridad» de `DESIGN.md`; controles nativos.

## 3. Backlog general (por prioridad)

| # | Tarea | Por qué | Estado |
|---|---|---|---|
| F | **Servidor de archivos** (sección 4) | Usar el XR como almacén privado accesible desde Mac y iPhone 17 Pro | ⬜ Próximo |
| 5 | Pruebas de calidad de la IA: 20–30 preguntas con respuesta esperada y un script que puntúe | Medir cualquier cambio de búsqueda o modelo | ⬜ |
| 4 | Reordenar fragmentos con un modelo pequeño de embeddings (~30 MB) | Acertar en preguntas generales | ⬜ (depende de 5) |
| 6 | Caché de respuestas repetidas; probar 1.5B en Q4_0 | Velocidad | ⬜ (depende de 5) |
| 9 | Script que sincronice los `.md` de los repos a Athena y Cyber 12 | Las copias se desactualizan | ⬜ |
| A | App del iPhone 17 Pro (cliente) | Necesaria para la fase F4 | ⬜ |
| — | Probar el acceso desde fuera de casa (datos móviles + Tailscale) | Validar disponibilidad real | ⬜ |

---

## 4. Servidor de archivos (diseño acordado: opción A)

### 4.1 Objetivo

Enviar documentos, imágenes o cualquier archivo desde el Mac o el iPhone 17 Pro, guardarlo **cifrado** en el XR y poder listarlo, verlo y descargarlo desde cualquiera de ellos, en casa o fuera (Tailscale).

### 4.2 Decisión de cifrado: «la llave la tiene el XR»

- Cada archivo se cifra **en el cliente** con la clave pública del Secure Enclave del XR (ECIES, como las notas). El XR guarda solo bultos cifrados.
- Para leer, el XR descifra en memoria y entrega por HTTPS fijado. Por eso **listar y descargar exigen la bóveda abierta**.
- Alternativa descartada (de momento): cifrado de extremo a extremo con llaves solo en los clientes. Más fuerte frente a un XR comprometido, pero complica añadir dispositivos y arriesga perderlo todo si se pierden los clientes.

### 4.3 Modelo de amenazas resumido

| Amenaza | Defensa |
|---|---|
| Robo del XR apagado o bloqueado | Bultos e índice cifrados; la llave no sale del Secure Enclave |
| Alguien en la red | HTTPS con certificado fijado + Tailscale |
| Token robado | Subir sí; listar/descargar exige bóveda abierta; borrar exige firma con Touch ID/Face ID |
| Orden repetida o alterada | Firma sobre método, ruta, hora, nonce y SHA-256 del cuerpo |
| Disco lleno o memoria agotada | 507 antes de aceptar; subida por trozos sin cargar el archivo entero en RAM |
| Coacción / emergencia | «Destrucción total»: borrar la llave del Secure Enclave deja todo irrecuperable al instante (fase F5) |

### 4.4 Formato en disco (XR)

```
Documents/AegisFiles/                     (isExcludedFromBackup = true, NSFileProtectionComplete)
├── index.aegis                           índice cifrado (ECIES), se reescribe atómicamente
├── blobs/<id>.blob                       contenido cifrado por trozos
├── thumbs/<id>.thumb                     miniatura cifrada (opcional)
└── uploads/<upload-id>/                  trozos en curso (se borran al confirmar o a las 24 h)
```

- **`id`**: 128 bits aleatorios en hex. No revela nombre ni tipo.
- **Contenido por trozos** (para archivos grandes sin cargarlos en RAM): cabecera ECIES una vez → clave de archivo de 256 bits; cada trozo de 4 MiB se cifra con ChaCha20-Poly1305 con nonce = prefijo aleatorio de 4 bytes ‖ contador de 8 bytes, y AAD = `id ‖ número de trozo ‖ es_último`. Detecta trozos reordenados, cortados o cambiados.
- **Índice** (JSON dentro de `index.aegis`): por archivo `id`, `name`, `mime`, `size`, `sha256` del claro, `created`, `folder`, `tags`, `has_thumb`, `uploader` (dispositivo emparejado).

### 4.5 API (todas con `Authorization: Bearer`, HTTPS fijado)

| Método | Ruta | Requisitos | Descripción |
|---|---|---|---|
| `POST` | `/v1/files/uploads` | token | Inicia subida: `{size, chunk_size}` → `{upload_id}`. Comprueba espacio libre (507). |
| `PUT` | `/v1/files/uploads/<upload_id>/<n>` | token | Trozo `n` ya cifrado + cabecera `X-Aegis-Chunk-SHA256`. Idempotente (reanudable). |
| `POST` | `/v1/files/uploads/<upload_id>/complete` | token | Envía la cabecera ECIES y los metadatos cifrados para el XR; mueve a `blobs/`. |
| `GET` | `/v1/files` | token + bóveda abierta | Lista (descifra el índice en RAM). Filtros `folder`, `q`. |
| `GET` | `/v1/files/<id>` | token + bóveda abierta | Descarga en streaming (descifra trozo a trozo). Soporta `Range`. |
| `GET` | `/v1/files/<id>/thumb` | token + bóveda abierta | Miniatura. |
| `PATCH` | `/v1/files/<id>` | token + bóveda abierta | Renombrar, mover de carpeta, etiquetas. |
| `DELETE` | `/v1/files/<id>` | token + **firma** | Borra bulto, miniatura y entrada del índice. |
| `GET` | `/v1/files/usage` | token | Espacio usado/libre y número de archivos (sin nombres). |

Notas de implementación:
- El servidor actual (`LocalHTTPServer`) lee la petición entera en memoria: **antes de F1 hay que añadir lectura del cuerpo en streaming** para los trozos y la descarga.
- Las rutas nuevas entran en `requiresMacSignature` cuando son destructivas (`DELETE /v1/files/*`).
- El índice descifrado vive solo en RAM mientras la bóveda está abierta y se borra con la purga.

### 4.6 Clientes

- **CLI del Mac:** `aegis_sync.py files put|get|ls|rm|mv` (cifra y trocea en el cliente; `rm` pide Touch ID).
- **Barra de menús:** pestaña «Archivos» con arrastrar y soltar, lista con miniaturas, descargar y borrar.
- **Finder (WebDAV):** servidor WebDAV mínimo sobre la misma API y TLS, para «Conectarse al servidor». Dificultad: Finder no admite certificados fijados → se instala un perfil de confianza solo para ese certificado o se usa un pequeño proxy local en el Mac que hace el fijado (preferido).
- **iPhone 17 Pro:** app cliente (SwiftUI, mismo `DESIGN.md`) con extensión **File Provider** para que el XR aparezca en la app Archivos; Face ID firma las órdenes con la clave de su Secure Enclave; emparejado con el mismo flujo de código.

### 4.7 Fases, tareas y criterios de terminado

**F0 — Base del servidor (prerrequisito)**
- [x] Lectura de cuerpo en streaming en `LocalHTTPServer` (sin cargar >1 trozo en RAM). *Hecho 2026-10-22: `AegisRequestBody` (trozos ≤256 KiB), `registerStreamingRoute`, 401/507 antes de leer el cuerpo, 413 (>16 MiB en rutas en RAM), 411, 431.*
- [x] Respuestas en streaming genéricas (ya existe para SSE; generalizar). *Hecho: `AegisStreamingResponse` + envío con contrapresión, también en SSE.*
- *Terminado cuando:* un `PUT` de 64 MiB no sube la huella de memoria más de ~10 MiB. ✅ *Prueba en Mac por TCP real: +0,3 MiB. En el XR, estado y sincronización siguen funcionando por HTTPS; la subida grande en el XR se probará con la primera ruta real (F1).*

**F1 — Núcleo de archivos en el XR**
- [x] `FileVaultStore` (Sources/FileVault): formato de bultos por trozos, índice cifrado, escritura atómica. *Hecho 2026-10-22. Cambio respecto a §4.4: en vez de un `index.aegis` único, cada archivo tiene su sobre `blobs/<id>.meta` (ECIES); así subir no exige descifrar nada y no hay un índice que reescribir. Protección de archivos «hasta el primer desbloqueo» (los bultos ya van cifrados y así se sirve con la pantalla bloqueada).*
- [x] Endpoints de 4.5 (subida reanudable, lista, descarga con Range, borrar firmado, usage). *Añadido `GET /v1/files/uploads/<id>` → trozos que faltan. La miniatura responde 404 hasta F2.*
- [x] Límite de espacio (507) y limpieza de subidas abandonadas. *Reserva fija de 1 GiB.*
- [x] Purga: borra el índice en RAM; no toca bultos.
- *Pruebas:* runner `FileVaultRunner` con subida/descarga redonda, trozo alterado (rechazado), reordenado (rechazado), reanudación, Range, borrado sin firma (403), bóveda cerrada (423).
- *Terminado cuando:* todas las pruebas pasan y en el XR real se sube y baja un PDF de 50 MB idéntico (SHA-256). ✅ *22/10/2026: `FileVaultRunner` 8/8. En el XR: subida 50 MB en 9,0 s (5,5 MB/s), bajada en 9,6 s (5,2 MB/s), SHA-256 idéntico, RAM estable (~192→195 MiB).*

**F2 — Mac (CLI + barra de menús)**
- [x] `files put/get/ls/rm/mv` en el CLI con barra de progreso y reanudación. *Hecho 2026-10-22 (`CompanionTools/aegis_files.py`): subida y bajada reanudables, SHA-256 verificado, miniaturas con QuickLook, clave pública del XR fijada en el primer uso, nombres en NFC.*
- [x] Pestaña «Archivos» en la barra de menús (arrastrar/soltar, miniaturas generadas en el Mac). *Carpetas navegables, «Subir…», ventana flotante para arrastrar (el panel se cierra al hacer clic fuera), descargar archivo o carpeta, borrar con Touch ID (carpeta: un solo Touch ID). La subida usa el CLI.*
- *Terminado cuando:* se sube una carpeta de fotos desde la barra de menús y se descarga desde el CLI. ✅ *22/10/2026: carpeta «Pruebas Gráficas» subida desde la barra y bajada con `files get --folder`. Arreglado de paso: el XR cortaba los últimos KB de algunas descargas TLS (24/24 correctas tras el arreglo).*

**F3 — Finder**
- [x] ~~Proxy local del Mac con fijado TLS + WebDAV de solo lectura; después escritura.~~ *Hecho (`files mount`), pero macOS 26.6 tarda ~90 s en montar cualquier WebDAV (fallo del sistema, comprobado también con wsgidav): queda como alternativa.*
- [x] Extensión **File Provider** (como iCloud Drive): «Aegis XR» en la barra lateral de Finder, sin montar discos. *Hecho 2026-10-22 (`MacApp/`, XcodeGen): descarga al abrir con SHA-256, subida cifrada en el Mac, renombrar/mover, sustituir oculta la versión anterior (`supersedes`, id estable `lineage`), borrar con un Touch ID, cambios del XR en ~20 s. Se activa una vez en Ajustes → Ítems de inicio y extensiones. Limitación aceptada: Finder borra antes de pedir Touch ID (avisa de que es permanente); si se cancela, macOS vuelve a pedirlo hasta confirmar.*
- *Terminado cuando:* el XR se monta en Finder y se abre un archivo con doble clic. ✅ *22/10/2026: doble clic abre documentos e imágenes (5–6 s la primera vez, luego desde caché); PDF real idéntico.*

**F4 — iPhone 17 Pro**
- [ ] App cliente: emparejamiento, estado, consultas a la IA, archivos.
- [ ] Extensión File Provider (Archivos) y «Compartir → Guardar en Aegis».
- [ ] Firma con Face ID (Secure Enclave del 17 Pro).
- *Terminado cuando:* desde datos móviles se guarda una foto en el XR y se abre desde Archivos.

**F5 — Extras**
- [ ] Indexar PDF/texto subidos para la IA (extracción de texto en el cliente).
- [ ] «Destrucción total» (borrar la llave del Secure Enclave, con doble confirmación firmada).
- [ ] Avisos al 17 Pro (XR desconectado, temperatura, purga).

### 4.8 Riesgos y límites asumidos

- iOS suspende apps en segundo plano: el XR debe estar enchufado, con EnclavR en primer plano (Acceso Guiado) y Tailscale activo. Tras reinicio o actualización hay que abrir la app.
- Capacidad = almacenamiento libre del XR menos iOS y modelos.
- Velocidad limitada por el Wi-Fi del XR y la conexión remota: apto para documentos y fotos, no para vídeo pesado diario.
- Batería: usar enchufe inteligente o rutina que mantenga 40–80 %.

---

## 5. Cómo retomar el trabajo (checklist)

1. `git pull` en `~/Desktop/EnclavR` y lee este plan y `.claude/napkin.md`.
2. Comprueba el XR: `aegis-status` (debe responder por HTTPS).
3. Compila y prueba en el Mac: `swift build` y los runners de `Tests/` (`.build/debug/*Runner`).
4. App del iPhone: `iOSApp/EnclavR.xcodeproj` (Team ID en `iOSApp/Signing.local.xcconfig`).
5. Antes de subir a GitHub: revisa que el diff no contenga IP reales, UDID, Team ID, rutas `/Users/`, tokens ni claves.
6. Al cerrar una fase: marca casillas, actualiza la sección 1 y documenta mediciones en `docs/ARQUITECTURA.md`.

## 6. Referencias en el repositorio

- Arquitectura técnica: `docs/ARQUITECTURA.md`
- Diseño de interfaz: `DESIGN.md`
- Puesta en marcha: `README.md` / `README_ES.md`
- Código clave: `Sources/NetworkGateway/LocalHTTPServer.swift` (rutas, firmas, emparejamiento), `Sources/NetworkGateway/TLSIdentityStore.swift`, `Sources/CoreCrypto/AegisBinaryCodec.swift` (formato ECIES), `CompanionTools/aegis_config.py` (TLS fijado y firmas en el Mac)
