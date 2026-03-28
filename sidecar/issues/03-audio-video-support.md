# Issue 3: Agregar Soporte Audio/Video

**Estado:** Pendiente  
**Prioridad:** Media  
**Estimación:** 2 días

---

## Descripción

Extender los parsers del sistema para soportar formatos de audio y video, integrando con Layer 0.5 (metadata nativa) y Layer 4 (LLM para contenido).

---

## Formatos Soportados

### Audio

| Formato | Librería | Campos |
|---------|----------|--------|
| MP3 | mutagen | artist, album, title, year, genre, track, duration |
| FLAC | mutagen | artist, album, title, year, genre, duration |
| WAV | mutagen | basic info |
| M4A/AAC | mutagen | artist, album, title, year, genre |

### Video

| Formato | Librería | Campos |
|---------|----------|--------|
| MP4 | ffprobe | title, duration, codec, resolution, fps |
| MKV | ffprobe | title, duration, codec |
| AVI | ffprobe | title, duration, codec |

---

## Metadata a Extraer

### Audio

```python
{
    "file_type": "audio",
    "format": "mp3",
    "bitrate": 320000,
    "duration": 245.5,  # segundos
    "artist": "The Beatles",
    "album": "Abbey Road",
    "title": "Come Together",
    "year": 1969,
    "genre": "Rock",
    "track_number": 1
}
```

### Video

```python
{
    "file_type": "video",
    "format": "mp4",
    "duration": 3600.0,
    "title": "Documental Historia",
    "codec": "h264",
    "resolution": "1920x1080",
    "fps": 30.0,
    "bitrate": 5000000
}
```

---

## Implementación

### 1. Audio Parser

```python
# sdk/parsers/audio_parser.py
from mutagen import File as MutagenFile
from pathlib import Path

class AudioParser(BaseParser):
    SUPPORTED_EXTENSIONS = ['.mp3', '.flac', '.wav', '.m4a', '.aac', '.ogg']
    
    def extract(self, file_path: str) -> dict:
        audio = MutagenFile(file_path)
        
        metadata = {
            "file_type": "audio",
            "format": Path(file_path).suffix[1:],
            "duration": audio.info.length if audio.info else None,
            "bitrate": audio.info.bitrate if audio.info else None,
        }
        
        # ID3 tags
        if hasattr(audio, 'tags'):
            tags = audio.tags
            metadata.update({
                "artist": tags.get('artist', [None])[0],
                "album": tags.get('album', [None])[0],
                "title": tags.get('title', [None])[0],
                "year": tags.get('date', [None])[0],
                "genre": tags.get('genre', [None])[0],
            })
        
        # Extraer texto para LLM
        text = self._build_text_content(metadata)
        
        return {
            "text": text,
            "metadata": metadata
        }
    
    def _build_text_content(self, metadata: dict) -> str:
        return f"""
Audio file: {metadata.get('title', 'Unknown')}
Artist: {metadata.get('artist', 'Unknown')}
Album: {metadata.get('album', 'Unknown')}
Year: {metadata.get('year', 'Unknown')}
Genre: {metadata.get('genre', 'Unknown')}
Duration: {metadata.get('duration', 0):.0f} seconds
        """.strip()
```

---

### 2. Video Parser

```python
# sdk/parsers/video_parser.py
import subprocess
import json
from pathlib import Path

class VideoParser(BaseParser):
    SUPPORTED_EXTENSIONS = ['.mp4', '.mkv', '.avi', '.mov', '.webm']
    
    def extract(self, file_path: str) -> dict:
        # Usar ffprobe para metadata
        cmd = [
            'ffprobe',
            '-v', 'quiet',
            '-print_format', 'json',
            '-show_format',
            '-show_streams',
            file_path
        ]
        
        result = subprocess.run(cmd, capture_output=True, text=True)
        data = json.loads(result.stdout)
        
        # Extraer información relevante
        format_info = data.get('format', {})
        streams = data.get('streams', [])
        
        video_stream = next((s for s in streams if s['codec_type'] == 'video'), {})
        
        metadata = {
            "file_type": "video",
            "format": Path(file_path).suffix[1:],
            "duration": float(format_info.get('duration', 0)),
            "title": format_info.get('tags', {}).get('title'),
            "codec": video_stream.get('codec_name'),
            "resolution": f"{video_stream.get('width', 0)}x{video_stream.get('height', 0)}",
            "fps": self._parse_fps(video_stream.get('r_frame_rate', '0/1')),
        }
        
        return {
            "text": self._build_text_content(metadata),
            "metadata": metadata
        }
    
    def _parse_fps(self, fps_str: str) -> float:
        """Parse '30000/1001' -> 29.97"""
        try:
            num, denom = fps_str.split('/')
            return float(num) / float(denom)
        except:
            return 0.0
    
    def _build_text_content(self, metadata: dict) -> str:
        return f"""
Video file: {metadata.get('title', 'Unknown')}
Duration: {metadata.get('duration', 0):.0f} seconds
Codec: {metadata.get('codec', 'Unknown')}
Resolution: {metadata.get('resolution', 'Unknown')}
        """.strip()
```

---

### 3. Registro de Parsers

```python
# sdk/parsers/__init__.py
from sdk.parsers.audio_parser import AudioParser
from sdk.parsers.video_parser import VideoParser

# Agregar al registry
PARSERS = {
    # ... existing
    ".mp3": AudioParser(),
    ".flac": AudioParser(),
    ".wav": AudioParser(),
    ".m4a": AudioParser(),
    ".mp4": VideoParser(),
    ".mkv": VideoParser(),
    ".avi": VideoParser(),
    ".mov": VideoParser(),
}
```

---

## Layer 0.5 para Audio/Video

La metadata nativa de audio/video tiene alta confiabilidad:

| Campo Presente | Score |
|----------------|-------|
| artist + title | +0.6 |
| album | +0.2 |
| year + genre | +0.2 |

**Threshold:** score > 0.6 → Layer 0.5 shortcut viable

---

## Dependencias a Agregar

```
# requirements.txt
mutagen>=1.47.0
ffmpeg-python>=0.2.0
```

---

## Tests Recomendados

- `test_audio_parser_mp3()` - MP3 con ID3
- `test_audio_parser_flac()` - FLAC tags
- `test_video_parser_mp4()` - MP4 con ffprobe
- `test_video_parser_fps_parsing()` - Edge cases FPS
- `test_layer_0_5_audio_shortcut()` - Audio con metadata completa

---

## Métricas Esperadas

| Métrica | Target |
|---------|--------|
| Formatos soportados | +8 (audio: 4, video: 4) |
| % audio/video con Layer 0.5 | >70% (ID3 usualmente completo) |
