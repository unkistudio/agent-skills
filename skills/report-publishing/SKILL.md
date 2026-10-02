---
name: report-publishing
description: Write a structured report.md and convert it into a polished, self-contained HTML file and then a PDF. Use whenever the user wants to produce a final report, write-up, or document for delivery — pentest reports, audit reports, status reports, project updates, documentation, meeting summaries, case studies, research notes, or any other written deliverable — especially when they mention converting Markdown to HTML/PDF, "make the final report", "produce a PDF of this", "make it presentable", or "I need to hand this in". Make sure to use this skill whenever the user says "report", "write-up", "deliverable", "final document", or asks for a formatted document, even if they don't explicitly say "PDF" or "HTML". It works for any type of report, not just security work.
---

# Report Publishing Workflow (Markdown → HTML → PDF)

This skill turns a structured Markdown report into a polished, self-contained HTML file and a print-ready PDF. It is domain-agnostic — it works for pentest reports, audit findings, status updates, project documentation, research notes, meeting minutes, or any other written deliverable. The workflow has three stages:

1. **Plan and write** the source `report.md` with a clear, consistent structure.
2. **Build branded HTML** from the Markdown with a converter script (PowerShell on Windows, Python on Linux/macOS — see OS selector below).
3. **Convert to PDF** using headless Edge/Chromium print-to-PDF.

### OS selector

Check the operating system first and follow the matching path through Stages 2–4:

- **Windows** → `build_html.ps1` (PowerShell 5.1+) and Edge via `cmd /c`.
- **Linux / macOS** → `build_html.py` (Python 3, standard library only) and any Chromium binary (Chrome, Brave, Chromium) via `subprocess`.

The Markdown source, styling, pagination rules, cover spec and verification criteria are identical on all systems — only the tooling blocks differ.

Work in the directory where the user wants the report (create one if the user asks, e.g. `Final_Report_<DATE>/`). Ask the user up front for: report title, audience (who reads it — affects tone and detail), language, any brand colours/logo, and whether they want the branded HTML/PDF at all or just the Markdown.

## Stage 1: Write the Markdown source

### 1.1 Gather requirements

Ask the user (do not assume):

- **Title and subject** of the report
- **Audience** — technical team, management, a client, the public. Tone and detail follow from this
- **Language** the report should be written in
- **Key sections** they need (see the template below for sensible defaults)
- **Branding** — a logo file path and brand colours, if they want it branded; otherwise use neutral styling
- **Deliverables** — Markdown only, or Markdown + HTML + PDF

### 1.2 Structure

Use this template as the default skeleton, dropping or renaming sections to fit the subject. Sections that don't apply should be removed — do not pad a report with empty headings.

```markdown
# <Report Title>

**Prepared for:** <Client/Recipient>
**Prepared by:** <Author>
**Date:** <DATE>
**Status:** Draft / Final
**Classification:** Confidential / Internal / Public

## Executive Summary

[2-3 paragraphs: what this report covers, the headline conclusion, the top 3-5 take-aways]

## Background

[Why this report exists, context the reader needs]

## Scope and Method

[What is covered, what is out of scope, how the information was gathered]

## Findings / Results / Analysis

### Finding 1: <Title>

- **Severity:** Critical / High / Medium / Low / Informational (omit for non-security reports)
- **Location/Asset:** <where it applies>
- **Description:** [What was found]
- **Evidence:** [Concrete details, data, quotes, log excerpts — not just references]
- **Impact:** [What it means for the reader]
- **Recommendation:** [How to address it]

[Repeat for each finding]

## Conclusion

[Recap of the key message and what happens next]

## Appendices

### A. Data Tables
### B. Source Documents / Evidence Index
### C. Definitions
```

### 1.3 Markdown style rules

- Use the headings hierarchy `#` title, `##` major sections, `###` subsections. Numbered top-level sections (`## 1. Executive Summary`) are also fine — they become hard page breaks in the PDF.
- Prefer **tables** for any comparative data (findings lists, metrics, statuses) — they render well in both HTML and PDF.
- Use `**bold**`, `*italic*`, and `` `code` `` inline. Link syntax `[text](url)` is stripped in the HTML build — put the URL as plain text if the reader needs it.
- Keep each finding self-contained: description, evidence, impact, recommendation.
- **Never invent evidence or data.** Only include what the user provided or what the underlying work actually produced. If something is missing, mark it clearly ("data not yet available") rather than fabricating it.

## Stage 2: Build the branded HTML

### Stage 2A (Windows)

Use a PowerShell converter script (Word COM `SaveAs` is unreliable — it hangs — avoid it entirely). Write `build_html.ps1` in the report folder. The script must:

1. **Parse the Markdown** and convert blocks to HTML — `#`/`##`/`###` headings, paragraphs, `ul`/`ol` lists, tables, fenced code blocks, blockquotes, horizontal rules — with an inline formatter handling `**bold**`, `` `code` ``, `*italic*`, and stripping `[text](url)` links to plain text. Two parser bugs were hit in practice and must be avoided:
   - **Skip table separator rows.** When collecting table lines, drop any row whose cells are all dashes (`| --- | --- |`) — otherwise it is emitted as a real `<tr><td>---</td></tr>` data row under the header.
   - **The colon ends up inside `<strong>`.** After inline formatting, `**Prepared for:** value` becomes `<strong>Prepared for:</strong> value`. If you extract the cover info table with a regex expecting `</strong>:`, nothing matches and the cover table comes out empty. Match the colon inside the tag: `<p><strong>([^<]+):</strong>\s*([^<]+)</p>`.
2. **Embed the logo** as a pre-scaled base64 data URI (scale to ~130 px wide first — a full-size logo in a header/footer renders huge). If the user provided no logo, skip the logo entirely and keep a clean text-only header.
3. **Wrap everything** in a single HTML file with an embedded `<style>` block.

Default styling (works for any report; adjust only if the user supplies brand colours):

- Headings: a strong display font, deep navy `#00113f`; body: a clean sans-serif in dark grey `#202020`; accents: electric blue `#0059cc`; light table fills `#ececec`. Load fonts via:

```css
@import url('https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:wght@400;600;700;800&family=Red+Hat+Display:wght@300;400;500;700&display=swap');
```

### Pagination rules that work in Chromium print-to-PDF

These are the mechanics that make the PDF come out right — they are verified on Edge/Chromium:

- `@page { size: A4; margin: 28mm 17mm 22mm 17mm; }` plus `@page :first { ... }` to drop the header/footer on the cover.
- Header/footer via **@page margin boxes** (`@top-left`, `@top-center`, `@bottom-center`, `@bottom-right`) — text and `counter(page)`/`counter(pages)` work; images work in margin boxes via `content: url(data:...)` **only if pre-scaled**. Logo in `@top-left`, report title in `@top-center`, footer "Confidential" (if applicable) + "Page X / Y".
- `h1,h2,h3,h4 { page-break-after: avoid; break-after: avoid; }` — no orphaned headings at the bottom of a page.
- Numbered top-level sections: `h2.sec { page-break-before: always; break-before: page; }`.
- Small tables (≤12 rows): tag as `table.keep { page-break-inside: avoid; break-inside: avoid; }` so a heading and its table stay together. Large tables must use `<thead>`/`<tbody>` so the header row repeats when split; `tr { page-break-inside: avoid; }`.
- Do **NOT** use `position: fixed` for header/footer logos — it lands inside the content in print. Use the margin boxes.

Cover page: top brand band, centred logo, report title, subtitle, info table (Prepared for / Author / Date / Status / Classification), and a short intro note.

### Stage 2B (Linux / macOS)

Write `build_html.py` in the report folder using **standard library only** (`re`, `html`, `base64`, `pathlib` — no pip dependencies so it runs anywhere). It must implement the same contract as Stage 2A:

1. **Parse the Markdown** with the same block coverage (headings, paragraphs, `ul`/`ol` lists, tables, fenced code blocks, blockquotes, horizontal rules) and the same inline formatting (`**bold**`, `` `code` ``, `*italic*`, strip `[text](url)` to plain text). Apply both parser-bug avoidances from Stage 2A (skip table separator rows; match the colon inside `<strong>` for the cover table).
2. **Embed the logo** as a base64 data URI. Prescale to ~130 px wide with PIL if it happens to be installed; otherwise embed the original file and cap its display size with CSS (`img { width: 130px; }`). Exception: logos inside `@page` margin-box headers ignore CSS sizing, so with no PIL available use a text-only running header instead of a margin-box logo.
3. **Wrap everything** in a single HTML file with the same embedded `<style>` block (same palette, fonts, cover structure and pagination rules as Stage 2A).

Write the HTML with `pathlib.Path.write_text(html, encoding="utf-8")` — no BOM handling needed on Linux/macOS. HTML-encode fenced code block contents with `html.escape`.

## Stage 3: Convert HTML to PDF (headless Chromium)

### Stage 3A (Windows — headless Edge)

Reliable Chromium conversion — no Word COM. **Do not use `Start-Process -ArgumentList` for this**: on Windows it has been observed to fail with "Multiple targets are not supported in headless mode" / exit code 13, because the array of arguments is not passed through as a single command line. The verified approach is to build one command string with every argument individually double-quoted and run it through `cmd /c`. `--no-first-run` avoids first-run dialogs in headless mode, and a unique `--user-data-dir` prevents a running Edge instance from hijacking the flags.

```powershell
$edge = "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"
$pdfPath = Join-Path $dir "report.pdf"
$url = "file:///" + ($htmlPath -replace '\\','/')
$profile = Join-Path $env:TEMP ("edgepdf_" + [guid]::NewGuid().ToString('N'))
$cmd = "`"$edge`" --headless --disable-gpu --no-first-run --no-pdf-header-footer --print-to-pdf=`"$pdfPath`" --user-data-dir=`"$profile`" `"$url`""
cmd /c $cmd 2>&1 | Out-String
Start-Sleep -Seconds 2
if (Test-Path $pdfPath) { (Get-Item $pdfPath).Length }
Remove-Item -Recurse -Force $profile -ErrorAction SilentlyContinue
```

Two notes on the output:
- The `cmd /c` run prints noisy-but-harmless Chromium errors (e.g. "Every renderer should have at least one task…" and occasional sync/network errors). Ignore them — judge success by the PDF file existing with a meaningful size.
- If the PDF does not appear, the most common causes are a missing `--no-first-run` or an Edge instance already running with the default profile; retry with a fresh `--user-data-dir`.

If Edge is not at that path, locate it via `Get-Command msedge` or check both `Program Files` and `Program Files (x86)` locations before failing.

### Stage 3B (Linux / macOS — headless Chromium)

Same flags, any Chromium binary. Locate one in this order: `google-chrome`, `chromium`, `chromium-browser`, `brave-browser` (via `shutil.which` / `command -v`), then the macOS app bundles (`/Applications/Google Chrome.app/Contents/MacOS/Google Chrome`, Brave and Chromium equivalents) before failing.

```python
import shutil, subprocess, tempfile
from pathlib import Path

binary = next((b for b in ("google-chrome", "chromium", "chromium-browser", "brave-browser")
               if shutil.which(b)), None)
pdf = Path(dir_) / "report.pdf"
profile = tempfile.mkdtemp(prefix="chromepdf_")
url = Path(html_path).as_uri()
cmd = [binary, "--headless", "--disable-gpu", "--no-first-run",
       "--no-pdf-header-footer", f"--print-to-pdf={pdf}",
       f"--user-data-dir={profile}", url]
res = subprocess.run(cmd, capture_output=True, text=True, timeout=120)
print(res.stderr[-2000:])  # noisy-but-harmless Chromium errors — see below
assert pdf.exists() and pdf.stat().st_size > 50_000, "PDF conversion failed"
shutil.rmtree(profile, ignore_errors=True)
```

The same two notes as Stage 3A apply: ignore the noisy Chromium stderr (judge success by the PDF existing with a meaningful size), and if no PDF appears the cause is almost always profile reuse — a fresh `--user-data-dir` fixes it.

### Practical notes (Windows)

- Save `.ps1` files as UTF-8 **with BOM** so PowerShell 5.1 parses accents and em-dashes correctly. Editors/write tools often produce UTF-8 without BOM, so explicitly re-add the BOM before running the script:
  ```powershell
  $p = "report-build\build_html.ps1"
  [IO.File]::WriteAllText($p, [IO.File]::ReadAllText($p, (New-Object System.Text.UTF8Encoding $true)), (New-Object System.Text.UTF8Encoding $true))
  ```
  Without the BOM, non-ASCII characters in the script (e.g. `×`, `—`) get misread by PowerShell 5.1.
- Write the HTML with `[IO.File]::WriteAllText($path, $html, (New-Object System.Text.UTF8Encoding $true))`.
- HTML-encode the contents of fenced code blocks (e.g. `[System.Net.WebUtility]::HtmlEncode`) — the report body often contains `<resource>`-style placeholders that would otherwise be parsed as tags.

## Stage 4: Verify

The model may not be able to read PDFs back (some models reject PDF input entirely), so verify the PDF programmatically rather than by eye. Windows (PowerShell) or cross-platform (Python) — same three checks:

```powershell
$bytes = [IO.File]::ReadAllBytes($pdfPath)
$ascii = [System.Text.Encoding]::ASCII.GetString($bytes)
"Pages: " + ([regex]::Matches($ascii, '/Type\s*/Page[^s]')).Count   # page count
"Images: " + ([regex]::Matches($ascii, '/Subtype\s*/Image')).Count  # logo XObjects (0 if no logo)
"Has EOF: " + $ascii.Contains('%%EOF')                              # file is complete
```

Linux / macOS (or any system with Python 3):

```python
import re
raw = Path(pdf_path).read_bytes().decode("ascii", errors="ignore")
print("Pages:", len(re.findall(r"/Type\s*/Page[^s]", raw)))    # page count
print("Images:", len(re.findall(r"/Subtype\s*/Image", raw)))   # logo XObjects (0 if no logo)
print("Has EOF:", "%%EOF" in raw)                              # file is complete
```

Then confirm the HTML by re-reading it:

- No leftover `position: fixed` header/footer logos, no orphaned headings.
- No `---` separator rows leaked into table bodies, and the cover info table is populated (not empty).
- Any summary tables are intact.

A few page-count rules of thumb: an empty cover table or a table full of `---` rows means the parser/meta extraction regressed; a tiny PDF (< 50 KB) usually means fonts did not embed.

## Important Rules

- **Ask before producing the branded HTML/PDF** if the user only asked for a Markdown report — or just produce all three if they said "report" without qualification, then tell them what was generated.
- **Do not fabricate content.** Write only what is supported by the user's input or the underlying work. Flag gaps instead of inventing data.
- **Keep the Markdown as the single source of truth.** The HTML and PDF are generated artifacts — never edit them by hand; if something changes, edit `report.md` and rebuild.
- **Follow the user's branding when provided**; fall back to the default palette otherwise. Match the user's requested language for all prose.
