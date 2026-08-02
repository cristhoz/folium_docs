# Prompt #3 — Fix de seguridad: validación real de DOCX/XLSX en adjuntos (Folium v1)

> Uso: pegar todo el contenido debajo de la línea como primer prompt en una sesión de Claude Code
> abierta en la **raíz del repositorio `folium`** (single-tenant, ya auditado con el prompt #2).
> Es de un solo tiro y de alcance acotado: corrige un único gap de seguridad, con tests, y no
> amplía funcionalidades.

---

Actúa como ingeniero de seguridad de **Folium v1** (gestión documental para una entidad pública
colombiana; Go + Chi + pgx + PostgreSQL, SPA embebida). El QA (`QA-INFORME.md`) dejó cero
hallazgos Críticos/Altos abiertos, pero **no cubrió** un gap de seguridad conocido en la
validación de adjuntos que viola la spec original ("validación MIME por magic bytes"). Tu tarea
es cerrarlo. Alcance estrictamente acotado: no toques nada fuera de lo descrito aquí.

## El problema

`internal/storage/mime.go` (`SniffMime`) detecta DOCX/XLSX solo por la firma ZIP
(`PK\x03\x04`) y devuelve `application/zip`; luego `resolveMime` en
`internal/handler/attachment.go` **confía en la extensión del nombre de archivo** para decidir
si ese ZIP es un `.docx` o un `.xlsx`:

```go
// internal/storage/mime.go:27-29
case bytes.HasPrefix(head, []byte{0x50, 0x4B, 0x03, 0x04}):
    // docx/xlsx are both zip containers; caller disambiguates via extension.
    return "application/zip"
```

Consecuencia: **cualquier archivo ZIP renombrado a `.docx`/`.xlsx` es aceptado** — un ZIP con
un `.exe` dentro, un JAR, un APK, un ZIP bomb, un ODT, lo que sea. El control de magic bytes
que sí funciona para PDF/JPEG/PNG/TIFF (verificado en QA: `.exe` renombrado a `.pdf` → 415) es
inexistente para los formatos Office. En una entidad pública que recibe documentos de
ciudadanos externos por ventanilla, esta es la superficie de entrada de malware más obvia del
sistema.

## La corrección exigida

### 1. Validación estructural OOXML (obligatoria)

Nueva función en `internal/storage` (junto a `SniffMime`), usando **solo stdlib**
(`archive/zip`, `encoding/xml`) — sin dependencias nuevas:

```go
// DetectOOXML valida que data sea un contenedor OOXML legítimo y devuelve su
// MIME type real ("" si no es un docx/xlsx válido).
func DetectOOXML(data []byte) string
```

Comportamiento requerido:

1. Abrir `data` como ZIP (`zip.NewReader` sobre `bytes.Reader`). Si no abre → `""`.
2. Localizar la entrada **`[Content_Types].xml`** (obligatoria en todo OOXML según ECMA-376).
   Si no existe → `""`.
3. Parsear su XML y buscar el content type del documento principal:
   - `application/vnd.openxmlformats-officedocument.wordprocessingml.document.main+xml`
     (Override para `/word/document.xml`) → es DOCX → devolver
     `application/vnd.openxmlformats-officedocument.wordprocessingml.document`.
   - `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet.main+xml`
     (Override para `/xl/workbook.xml`) → es XLSX → devolver
     `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`.
4. Verificar que la parte principal declarada (`word/document.xml` o `xl/workbook.xml`)
   **existe realmente como entrada del ZIP** — un `[Content_Types].xml` copiado dentro de un
   ZIP arbitrario no debe bastar.
5. **Rechazar explícitamente** (devolver `""`) todo lo demás, en particular:
   - Formatos con macros: `.docm`/`.xlsm` (`application/vnd.ms-word.document.macroEnabled.main+xml`,
     `application/vnd.ms-excel.sheet.macroEnabled.main+xml`) — son vectores clásicos de malware
     y NO están en `AllowedMimeTypes`.
   - Plantillas (`.dotx`, `.xltx`), ODF renombrado, JAR/APK (ZIP con `META-INF/`), y cualquier
     ZIP genérico.

### 2. Defensas al parsear (el validador no puede ser el nuevo vector)

- **Nunca descomprimir entradas distintas de `[Content_Types].xml`** — la verificación del
  punto 4 es solo por nombre de entrada en el directorio central, sin extraer.
- Cap de lectura al descomprimir `[Content_Types].xml`: máximo **1 MiB** descomprimido
  (`io.LimitReader`); si se excede → `""` (protección contra ZIP bomb dirigido al validador).
- Cap de entradas del ZIP a recorrer: si el archivo declara más de **10 000 entradas** → `""`.
- El validador opera sobre el buffer completo ya en memoria (el handler ya bufferea el archivo
  entero en `buf` antes de guardar, y `MaxBytesReader` + `MAX_FILE_SIZE_MB` acotan su tamaño) —
  no cambies ese modelo ni escribas a disco antes de validar.

### 3. Integración en el handler

En `internal/handler/attachment.go` (`handleUploadAttachment`):

- Mantén el sniff barato de los primeros 512 bytes como **primer filtro** (rechazo rápido de
  todo lo que no tenga firma conocida, igual que hoy).
- Cuando `SniffMime` devuelva `application/zip`: **completar la lectura del archivo al buffer
  y solo entonces** llamar `DetectOOXML(buf.Bytes())`. El MIME type resultante debe además
  **coincidir con la extensión** del nombre (`.docx` ↔ wordprocessingml, `.xlsx` ↔
  spreadsheetml); un XLSX real renombrado a `.docx` → rechazo (el nombre con que se sirve el
  archivo después no debe mentir sobre su contenido).
- Todo rechazo del validador → **415** con code `unsupported_type` (mismo contrato que hoy).
- El flujo de PDF/JPEG/PNG/TIFF no cambia en nada. Los códigos 413 (tamaño) y 400 (sin
  archivo) no cambian. El sha256 se sigue calculando sobre el contenido completo.
- Elimina o adapta `resolveMime`: la extensión deja de ser fuente de verdad para OOXML.
- Actualiza el comentario engañoso de `mime.go` ("caller disambiguates via extension").

### 4. Endurecimiento mínimo al servir (mismo commit o commit aparte)

En `serveAttachment` (`internal/handler/attachment.go`):

- Añadir `X-Content-Type-Options: nosniff` a las respuestas de view/download.
- Escapar correctamente el filename en `Content-Disposition` usando
  `mime.FormatMediaType("attachment", map[string]string{"filename": ...})` en vez de la
  concatenación manual con comillas (hoy un `OriginalName` con `"` o CR/LF malforma el header).

No agregues nada más al serving (no antivirus, no re-encodado, no CSP de adjuntos): fuera de
alcance.

## Tests exigidos (sin fixtures binarios en el repo — genera los archivos en el test)

Unit tests en `internal/storage` construyendo los ZIPs en memoria con `archive/zip.Writer`:

1. DOCX mínimo válido (`[Content_Types].xml` con el Override correcto + `word/document.xml`) →
   MIME de DOCX.
2. XLSX mínimo válido → MIME de XLSX.
3. ZIP genérico (un `hola.txt`) → `""`.
4. ZIP **sin** `[Content_Types].xml` → `""`.
5. `[Content_Types].xml` correcto pero **sin** la parte `word/document.xml` en el ZIP → `""`.
6. Content type de DOCM (macroEnabled) → `""`.
7. `[Content_Types].xml` que descomprime a más de 1 MiB → `""` (sin OOM ni cuelgue).
8. No-ZIP (bytes arbitrarios) → `""`.

Test de handler en `internal/handler` (mismo estilo que `TestLoginCookieSecureFollowsConfig`):

9. Upload multipart de un ZIP genérico renombrado `.docx` → **415** `unsupported_type`.
10. Upload de un DOCX mínimo válido → **201**.
11. Upload de un XLSX válido renombrado `.docx` (extensión cruzada) → **415**.

## Verificación y criterio de terminado

- `go build ./...`, `go vet ./...`, `golangci-lint run ./...`, `go test ./... -count=1 -p 1`
  (con `TEST_DATABASE_URL` si los tests de integración lo requieren) — todo en verde, incluidos
  los tests nuevos.
- Re-verificación en vivo contra `docker compose up --build`: los tres casos de handler de
  arriba vía curl con una sesión real, más una subida de PDF válido (sin regresión, 201).
- **Commits:** Conventional Commits, referenciando este fix como `QA-008` (continúa la
  numeración del informe). Uno para la validación OOXML (+su integración y tests), otro para el
  endurecimiento del serving si prefieres separarlo.
- Actualiza `QA-INFORME.md`: añade el hallazgo `QA-008 — Alto` (con la evidencia del gap y su
  corrección, formato de los hallazgos existentes) y súmalo a la sección de verificación final.
  Actualiza `DECISIONES.md` si documenta la validación de adjuntos.
- **No toques** los cambios sin commitear que existen en `web/` (refactor de Login en curso):
  no los incluyas en ningún commit (`git add` selectivo, nunca `git add -A` / `-a`) y no los
  modifiques ni los revirtas.
- No amplíes `AllowedMimeTypes`, no agregues dependencias, no cambies el contrato de la API.
