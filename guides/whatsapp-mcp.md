# WhatsApp MCP para Claude Code

Guía completa para conectar tu WhatsApp personal a Claude Code: leer mensajes, descargar audios y responder, todo desde el chat.

---

## Qué instala este setup

| Componente | Qué hace |
|---|---|
| **whatsapp-bridge** (Go) | Se conecta a WhatsApp Web, sincroniza mensajes en SQLite, expone REST API en `localhost:8080` |
| **whatsapp-mcp-server** (Python) | Traduce las tools de MCP a consultas SQL + llamadas REST al bridge |
| **openai-whisper** | Transcripción de audios en español (calidad superior a MarkItDown para voz) |

Repo base: [lharries/whatsapp-mcp](https://github.com/lharries/whatsapp-mcp) (~5.5k ★)

---

## Prerequisitos

```bash
brew install go uv ffmpeg
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

> **Fix necesario en main.go**: la versión actual de whatsmeow requiere `context.Background()` como primer argumento en 5 funciones. Aplicar estos cambios:
> ```go
> client.Download(context.Background(), downloader)          // línea ~644
> sqlstore.New(context.Background(), "sqlite3", ...)         // línea ~803
> container.GetFirstDevice(context.Background())             // línea ~810
> client.GetGroupInfo(context.Background(), jid)             // línea ~976
> client.Store.Contacts.GetContact(context.Background(), jid) // línea ~991
> ```

### 2. Registrar el MCP en Claude Code

```bash
claude mcp add whatsapp -- /opt/homebrew/bin/uv run \
  --directory ~/Development/whatsapp-mcp/whatsapp-mcp-server main.py
```

### 3. Fix de serialización del MCP server

El servidor Python retorna dataclasses que FastMCP no puede serializar a JSON. Agregar en `whatsapp-mcp-server/main.py`:

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

Y envolver cada `return` de las tools con `_serialize()`:
```python
return _serialize(whatsapp_list_chats(...))
return _serialize(whatsapp_search_contacts(...))
# etc. para todas las funciones que retornan Chat, Message, Contact, MessageContext
```

### 4. Symlink de la base de datos

El bridge guarda la DB donde lo ejecutás. Si lo corrés desde `~/`, la DB queda en `~/store/`. El MCP server Python la busca en `../whatsapp-bridge/store/`. Solucionarlo con un symlink:

```bash
# Ejecutar UNA vez, después del primer inicio del bridge
ln -sf ~/store ~/Development/whatsapp-mcp/whatsapp-bridge/store
```

---

## Uso diario

### Arrancar el bridge (terminal separada)

```bash
# IMPORTANTE: correrlo desde el home (~/) para que la DB quede siempre en el mismo lugar
cd ~
~/Development/whatsapp-mcp/whatsapp-bridge/whatsapp-bridge
```

La primera vez muestra un QR. Escanearlo con WhatsApp → Configuración → Dispositivos vinculados → Vincular dispositivo.

Las siguientes veces conecta automáticamente.

### Reiniciar Claude Code

Reiniciar para que cargue el MCP de WhatsApp. Las tools disponibles son:

| Tool MCP | Qué hace |
|---|---|
| `list_chats` | Lista conversaciones recientes |
| `search_contacts` | Busca un contacto por nombre o número |
| `list_messages` | Lee mensajes de un chat (filtrable por fecha, contacto, texto) |
| `send_message` | Envía un mensaje de texto |
| `send_file` | Envía imagen, video o documento |
| `send_audio_message` | Envía un audio (convierte a .ogg automáticamente) |
| `download_media` | Descarga media de un mensaje |
| `get_chat` | Metadata de un chat por JID |

---

## Descargar audios (workaround)

El `download_media` del MCP puede dar 403 con versiones nuevas de whatsmeow. En ese caso, descargar y descifrar directamente:

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
        "WHERE chat_jid=? AND media_type='audio' ORDER BY timestamp",
        (chat_jid,)
    ).fetchall()
    conn.close()
    for i, (msg_id, ts, from_me, url, key) in enumerate(rows):
        who = "yo" if from_me else "ellos"
        ts_clean = str(ts).replace(" ","_").replace(":","-")
        out = f"{out_dir}/{i+1:02d}_{ts_clean}_{who}.ogg"
        enc = urllib.request.urlopen(
            urllib.request.Request(url, headers={"User-Agent": "WhatsApp/2.26.5.75 A"})
        ).read()
        with open(out, "wb") as f:
            f.write(decrypt_whatsapp_media(enc, key, "audio"))
        print(f"[{i+1}/{len(rows)}] {out}")
```

> Instalar: `pip3 install cryptography`

---

## Transcribir audios en español

MarkItDown (docmd) da resultados pobres en español. Usar Whisper:

```bash
pip3 install openai-whisper
# Convertir OGG a MP3 primero
for f in audios/*.ogg; do ffmpeg -y -i "$f" "${f%.ogg}.mp3" 2>/dev/null; done
```

```python
import whisper
model = whisper.load_model("small")  # descarga ~461MB la primera vez

def transcribir(mp3_path: str) -> str:
    result = model.transcribe(mp3_path, language="es", fp16=False)
    return result["text"].strip()
```

> `language="es"` es clave para español rioplatense.  
> `fp16=False` necesario en Apple Silicon sin GPU dedicada.

---

## Buscar contactos con JID @lid

WhatsApp nuevo usa JIDs tipo `@lid` en vez del número para algunos contactos. Si `search_contacts` no encuentra a alguien, buscar directamente:

```bash
sqlite3 ~/store/whatsapp.db \
  "SELECT their_jid, full_name FROM whatsmeow_contacts WHERE their_jid LIKE '%NUMERO%' OR full_name LIKE '%NOMBRE%'"
```

El JID resultante (ej. `214039795875987@lid`) es el que usar en `list_messages`, `download_media`, etc.

---

## Estructura de archivos

```
~/store/
  messages.db     ← mensajes sincronizados (texto + metadata de media)
  whatsapp.db     ← sesión WhatsApp + contactos

~/Development/whatsapp-mcp/
  whatsapp-bridge/
    whatsapp-bridge   ← binario Go (el bridge)
    store → ~/store   ← symlink
  whatsapp-mcp-server/
    main.py           ← MCP server (con fix de serialización aplicado)
    whatsapp.py       ← queries SQL a messages.db
```

---

## Troubleshooting

| Síntoma | Causa | Solución |
|---|---|---|
| Tools retornan vacío | Bug de serialización dataclasses | Aplicar fix `_serialize()` en main.py |
| `download_media` da 403 | Incompatibilidad whatsmeow nuevo | Usar script de descifrado directo |
| Bridge: `client outdated (405)` | whatsmeow desactualizado | `go get go.mau.fi/whatsmeow@latest && go build` |
| Contacto no aparece en `search_contacts` | Usa JID @lid | Buscar en `whatsapp.db` tabla `whatsmeow_contacts` |
| DB no encontrada por MCP server | Bridge corrido desde directorio incorrecto | Crear symlink o correr bridge desde `~/` |
