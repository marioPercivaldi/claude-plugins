# WhatsApp MCP para Claude Code

Guía completa para conectar tu WhatsApp personal a Claude Code: leer mensajes, descargar audios y responder, todo desde el chat.

---

## Cómo funciona

```
Claude Code
    └── MCP server (Python) — arranca automáticamente al iniciar Claude Code
            └── whatsapp-bridge (Go) — arranca automáticamente si no está corriendo
                    └── WhatsApp Web ← sincroniza mensajes, expone REST en localhost:8080
```

**El bridge arranca solo.** No hace falta abrir ninguna terminal ni correr nada manualmente. La única excepción es la primera vez (escaneo del QR de vinculación).

---

## Componentes

| Componente | Qué hace |
|---|---|
| **whatsapp-bridge** (Go) | Se conecta a WhatsApp Web, sincroniza mensajes en SQLite, expone REST API en `localhost:8080` |
| **whatsapp-mcp-server** (Python) | Traduce las tools de MCP a consultas SQL + llamadas REST al bridge. **Auto-inicia el bridge si no está corriendo.** |
| **openai-whisper** | Transcripción de audios en español (calidad superior a MarkItDown para voz) |

Repo base: [lharries/whatsapp-mcp](https://github.com/lharries/whatsapp-mcp) (~5.5k ★)

---

## Prerequisitos

```bash
brew install go uv ffmpeg
pip3 install cryptography openai-whisper
```

---

## Instalación

### 1. Clonar y compilar el bridge

```bash
cd ~/Development
git clone https://github.com/lharries/whatsapp-mcp.git
cd whatsapp-mcp/whatsapp-bridge

# Actualizar whatsmeow a la última versión (la del repo suele estar desactualizada)
go get go.mau.fi/whatsmeow@latest
go get go.mau.fi/util/dbutil@latest
go build -o whatsapp-bridge .
```

> **Fix necesario en main.go**: la versión actual de whatsmeow requiere `context.Background()` como primer argumento en 5 funciones:
> ```go
> client.Download(context.Background(), downloader)           // línea ~644
> sqlstore.New(context.Background(), "sqlite3", ...)          // línea ~803
> container.GetFirstDevice(context.Background())              // línea ~810
> client.GetGroupInfo(context.Background(), jid)              // línea ~976
> client.Store.Contacts.GetContact(context.Background(), jid) // línea ~991
> ```

### 2. Primera vinculación (una sola vez)

Correr el bridge manualmente para escanear el QR:

```bash
cd ~
~/Development/whatsapp-mcp/whatsapp-bridge/whatsapp-bridge
```

Escanear con WhatsApp → Configuración → Dispositivos vinculados → Vincular dispositivo.

Una vez vinculado, **ya no hace falta volver a hacer esto**. La sesión queda guardada en `~/store/whatsapp.db`.

### 3. Symlink de la base de datos (una sola vez)

El bridge guarda la DB en `~/store/` (relativo a donde se ejecuta). El MCP server la busca en `../whatsapp-bridge/store/`:

```bash
ln -sf ~/store ~/Development/whatsapp-mcp/whatsapp-bridge/store
```

### 4. Registrar el MCP en Claude Code

```bash
claude mcp add whatsapp -- /opt/homebrew/bin/uv run \
  --directory ~/Development/whatsapp-mcp/whatsapp-mcp-server main.py
```

### 5. Fixes al MCP server (main.py)

**Fix 1 — Auto-start del bridge:**

```python
import subprocess, time, threading, requests
from pathlib import Path

BRIDGE_BINARY = Path(__file__).parent.parent / "whatsapp-bridge" / "whatsapp-bridge"
BRIDGE_WORKDIR = Path.home()
BRIDGE_API     = "http://localhost:8080/api/send"
BRIDGE_LOG     = Path.home() / "store" / "bridge.log"

def _bridge_running() -> bool:
    try:
        requests.post(BRIDGE_API, json={}, timeout=2)
        return True
    except Exception:
        return False

def _ensure_bridge() -> str | None:
    if _bridge_running():
        return None
    BRIDGE_LOG.parent.mkdir(parents=True, exist_ok=True)
    subprocess.Popen(
        [str(BRIDGE_BINARY)], cwd=str(BRIDGE_WORKDIR),
        stdout=open(BRIDGE_LOG, "a"), stderr=subprocess.STDOUT,
    )
    for _ in range(10):
        time.sleep(1)
        if _bridge_running():
            return None
    return (
        "WhatsApp bridge started but not connected. "
        "First time? Run manually to scan QR: ~/Development/whatsapp-mcp/whatsapp-bridge/whatsapp-bridge"
    )

# Arrancar bridge en background cuando carga el MCP server
threading.Thread(target=_ensure_bridge, daemon=True).start()
```

**Fix 2 — Serialización de dataclasses:**

```python
import dataclasses
from datetime import datetime, date

def _serialize(obj):
    if dataclasses.is_dataclass(obj) and not isinstance(obj, type):
        return {k: _serialize(v) for k, v in dataclasses.asdict(obj).items()}
    if isinstance(obj, (datetime, date)):
        return obj.isoformat()
    if isinstance(obj, list):
        return [_serialize(i) for i in obj]
    if isinstance(obj, dict):
        return {k: _serialize(v) for k, v in obj.items()}
    return obj
```

Envolver cada `return` de las tools con `_serialize()`, y en `send_message`, `send_file`, `send_audio_message`, `download_media` agregar:
```python
err = _ensure_bridge()
if err:
    return {"success": False, "message": err}
```

### 6. Reiniciar Claude Code

Reiniciar para que cargue el MCP. A partir de ahora el bridge arranca solo.

---

## Tools disponibles

| Tool | Qué hace |
|---|---|
| `list_chats` | Lista conversaciones recientes |
| `search_contacts` | Busca contacto por nombre o número |
| `list_messages` | Lee mensajes (filtrable por fecha, contacto, texto) |
| `send_message` | Envía texto |
| `send_file` | Envía imagen, video o documento |
| `send_audio_message` | Envía audio (convierte a .ogg automáticamente) |
| `download_media` | Descarga media de un mensaje |
| `get_chat` | Metadata de un chat por JID |

---

## Descargar audios (workaround al 403)

El `download_media` del MCP puede dar 403 con versiones nuevas de whatsmeow. Script alternativo:

```python
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from cryptography.hazmat.primitives.hashes import SHA256
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend
import sqlite3, urllib.request, os

DB_PATH = os.path.expanduser("~/store/messages.db")

def decrypt_whatsapp_media(encrypted: bytes, media_key: bytes, media_type: str) -> bytes:
    info = {
        "audio": b"WhatsApp Audio Keys",
        "image": b"WhatsApp Image Keys",
        "video": b"WhatsApp Video Keys",
        "document": b"WhatsApp Document Keys",
    }[media_type]
    expanded = HKDF(SHA256(), 112, None, info, default_backend()).derive(media_key)
    iv, aes_key = expanded[:16], expanded[16:48]
    ct = encrypted[:-10]
    dec = Cipher(algorithms.AES(aes_key), modes.CBC(iv), default_backend()).decryptor()
    pt = dec.update(ct) + dec.finalize()
    return pt[:-pt[-1]] if pt[-1] < 16 else pt

def download_audios(chat_jid: str, out_dir: str):
    os.makedirs(out_dir, exist_ok=True)
    conn = sqlite3.connect(DB_PATH)
    rows = conn.execute(
        "SELECT id, timestamp, is_from_me, url, media_key FROM messages "
        "WHERE chat_jid=? AND media_type='audio' ORDER BY timestamp", (chat_jid,)
    ).fetchall()
    conn.close()
    for i, (msg_id, ts, from_me, url, key) in enumerate(rows):
        who = "yo" if from_me else "ellos"
        out = f"{out_dir}/{i+1:02d}_{str(ts).replace(' ','_').replace(':','-')}_{who}.ogg"
        enc = urllib.request.urlopen(
            urllib.request.Request(url, headers={"User-Agent": "WhatsApp/2.26.5.75 A"})
        ).read()
        with open(out, "wb") as f:
            f.write(decrypt_whatsapp_media(enc, key, "audio"))
        print(f"[{i+1}/{len(rows)}] {out}")
```

---

## Transcribir audios en español

MarkItDown da resultados malos en español. Usar Whisper:

```bash
pip3 install openai-whisper
for f in audios/*.ogg; do ffmpeg -y -i "$f" "${f%.ogg}.mp3" 2>/dev/null; done
```

```python
import whisper
model = whisper.load_model("small")  # ~461MB, se descarga una sola vez

def transcribir(mp3_path: str) -> str:
    return model.transcribe(mp3_path, language="es", fp16=False)["text"].strip()
```

> `language="es"` es clave para español rioplatense. `fp16=False` necesario en Apple Silicon.

---

## Buscar contacto cuando no aparece (JID @lid)

WhatsApp nuevo usa JIDs `@lid` en vez del número para algunos contactos:

```bash
sqlite3 ~/store/whatsapp.db \
  "SELECT their_jid, full_name FROM whatsmeow_contacts WHERE their_jid LIKE '%NUMERO%' OR full_name LIKE '%NOMBRE%'"
```

Usar el JID resultante (ej. `214039795875987@lid`) en cualquier tool.

---

## Estructura de archivos

```
~/store/
  messages.db     ← mensajes sincronizados
  whatsapp.db     ← sesión + contactos
  bridge.log      ← log del bridge (auto-start)

~/Development/whatsapp-mcp/
  whatsapp-bridge/
    whatsapp-bridge   ← binario Go
    store → ~/store   ← symlink (necesario)
  whatsapp-mcp-server/
    main.py           ← MCP server (fixes aplicados)
    whatsapp.py       ← queries SQL
```

---

## Troubleshooting

| Síntoma | Causa | Solución |
|---|---|---|
| Tools retornan vacío | Serialización dataclasses | Aplicar fix `_serialize()` en main.py |
| `download_media` da 403 | Incompatibilidad whatsmeow nuevo | Usar script de descifrado directo |
| Bridge: `client outdated (405)` | whatsmeow desactualizado | `go get go.mau.fi/whatsmeow@latest && go build` |
| Bridge no conecta al auto-arrancar | Primera vez, necesita QR | Correr bridge manualmente una vez |
| Contacto no aparece en `search_contacts` | JID @lid | Buscar en `whatsapp.db` → `whatsmeow_contacts` |
| DB no encontrada | Bridge corrido desde directorio incorrecto | Crear symlink `~/Development/whatsapp-mcp/whatsapp-bridge/store → ~/store` |
