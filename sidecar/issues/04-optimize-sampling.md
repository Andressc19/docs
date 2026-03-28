# Issue 4: Optimizar Sampling para Archivos Grandes

**Estado:** Pendiente  
**Prioridad:** Media  
**Estimación:** 1 día

---

## Descripción

Limitar la cantidad de contenido enviado al LLM para archivos grandes (PDFs, Excel, imágenes) mediante muestreo inteligente, reduciendo tokens y latencia.

---

## Problema

Archivos grandes generan mucho contenido → más tokens → mayor costo y latencia.

| Tipo | Tamaño | Contenido | Tokens Est. |
|------|--------|------------|--------------|
| PDF 100 páginas | 5MB | 50,000 palabras | ~60,000 |
| Excel 50 sheets | 2MB | 10,000 celdas | ~20,000 |
| Imagen 4K | 10MB | pixels | ~1,000,000 (vision) |

---

## Estrategias de Sampling

### PDF: Muestreo de 3 Páginas

**Estrategia:** Extraer inicio, medio y fin del documento.

```python
# sdk/parsers/pdf_parser.py
class PdfParser(BaseParser):
    def extract(self, file_path: str, max_pages: int = 3) -> dict:
        reader = PdfReader(file_path)
        total_pages = len(reader.pages)
        
        if total_pages <= max_pages:
            # Documento pequeño, extraer todo
            return self._extract_all(reader)
        
        # Sampling: inicio, medio, fin
        page_indices = [
            0,  # Primera página
            total_pages // 2,  # Página media
            total_pages - 1  # Última página
        ]
        
        sampled_text = []
        for idx in page_indices:
            page = reader.pages[idx]
            text = page.extract_text()
            sampled_text.append(f"[Página {idx + 1}]\n{text}")
        
        return {
            "text": "\n\n".join(sampled_text),
            "metadata": {
                "total_pages": total_pages,
                "sampled_pages": len(page_indices),
                "sampling_method": "3-point"
            }
        }
```

**Configuración opcional:**
```python
# CLI
--pdf-sampling pages  # 3 páginas (default)
--pdf-sampling full   # documento completo
--pdf-sampling smart   # detecta estructura y muestrea headers
```

---

### Excel: Muestreo por Sheet

**Estrategia:** Solo primeras 5 filas de cada sheet.

```python
# sdk/parsers/xlsx_parser.py
class XlsxParser(BaseParser):
    def extract(self, file_path: str, max_rows: int = 5) -> dict:
        wb = load_workbook(file_path, read_only=True)
        
        sampled_data = {}
        for sheet_name in wb.sheetnames:
            sheet = wb[sheet_name]
            
            # Headers (primera fila)
            headers = [cell.value for cell in next(sheet.iter_rows())]
            
            # Primeras N filas de datos
            rows = []
            for i, row in enumerate(sheet.iter_rows()):
                if i >= max_rows:
                    break
                rows.append([cell.value for cell in row])
            
            sampled_data[sheet_name] = {
                "headers": headers,
                "rows": rows
            }
        
        text = self._build_text(sampled_data)
        
        return {
            "text": text,
            "metadata": {
                "total_sheets": len(wb.sheetnames),
                "sampled_rows_per_sheet": max_rows,
                "sampling_method": "top-n"
            }
        }
    
    def _build_text(self, data: dict) -> str:
        lines = []
        for sheet_name, content in data.items():
            lines.append(f"[Sheet: {sheet_name}]")
            lines.append(f"Headers: {content['headers']}")
            for row in content['rows']:
                lines.append(str(row))
            lines.append("")
        return "\n".join(lines)
```

---

### Imágenes: Thumbnail en lugar de Original

**Estrategia:** Redimensionar antes de enviar a visión.

```python
# sdk/parsers/image_parser.py
class ImageParser(BaseParser):
    MAX_DIMENSION = 1024  # Max 1024px en cualquier lado
    
    def extract(self, file_path: str) -> dict:
        img = Image.open(file_path)
        
        # Generar thumbnail si es muy grande
        if max(img.size) > self.MAX_DIMENSION:
            img.thumbnail((self.MAX_DIMENSION, self.MAX_DIMENSION))
            # Guardar thumbnail temporal para enviar a LLM
            temp_path = self._save_thumbnail(img, file_path)
            thumbnail_path = temp_path
        else:
            thumbnail_path = file_path
        
        # Extraer EXIF (para Layer 0.5)
        exif = self._extract_exif(img)
        
        return {
            "text": self._build_text(exif),
            "thumbnail_path": thumbnail_path,  # Enviar a LLM
            "metadata": {
                "original_size": img.size,
                "is_thumbnail": thumbnail_path != file_path
            }
        }
```

---

## Configuración Global

```python
# sdk/config.py
class SamplingConfig:
    pdf_max_pages: int = 3
    pdf_method: str = "3-point"  # 3-point, smart, full
    
    xlsx_max_rows: int = 5
    xlsx_method: str = "top-n"  # top-n, all
    
    image_max_dimension: int = 1024
    image_quality: int = 85
```

```bash
# CLI
--sampling strict   # Máxima optimización
--sampling balanced # Default
--sampling full     # Sin sampling
```

---

## Métricas de Impacto

| Archivo | Tokens (sin sampling) | Tokens (con sampling) | Reducción |
|---------|----------------------|----------------------|-----------|
| PDF 100p | ~60,000 | ~3,000 | 95% |
| Excel 50s | ~20,000 | ~1,000 | 95% |
| Imagen 4K | ~1,000,000 | ~50,000 | 95% |

---

## Tests Recomendados

- `test_pdf_sampling_3_pages()` - PDF de 100 páginas
- `test_pdf_sampling_small_doc()` - PDF de 2 páginas (no sampling)
- `test_xlsx_sampling()` - Excel con 10 sheets
- `test_image_thumbnail_generation()` - Imagen 4K → thumbnail
- `test_sampling_config_options()` - CLI options

---

## Consideraciones

### Precision vs Costo

- Sampling puede perder información en medio del documento
- Para contratos/legal: usar sampling "smart" (detectar secciones)
- Para análisis general: sampling default es suficiente

### Edge Cases

- PDF con imágenesscaneadas → OCR necesario
- Excel con fórmulas → valores vs fórmulas
