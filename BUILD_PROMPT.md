# Full Claude Code Prompt: Build 3b1b-calculus Site

Run this with:
```bash
cd /Users/yuechen/home/sophie/3b1b-calculus
claude --dangerously-skip-permissions -p "$(cat BUILD_PROMPT.md)"
```

---

## Task

Build an interactive Quarto website for 3Blue1Brown's "Essence of Calculus" video series. Download videos from YouTube, transcribe them, generate rich lesson pages with interactive elements, and deploy to GitHub Pages.

## Video Source

Download the full "Essence of Calculus" playlist from 3Blue1Brown's YouTube channel. Start with Chapter 1:
- https://www.youtube.com/watch?v=WUvTyaaNkzM (Chapter 1: The essence of calculus)

Use `yt-dlp` to download. Install if needed: `brew install yt-dlp`

The full playlist URL: https://www.youtube.com/playlist?list=PLZHQObOWTQDMsr9K-rj53DwVRMYO3t5Yr

Download all videos in the playlist to `videos/` directory:
```bash
yt-dlp -f 'bestvideo[ext=mp4]+bestaudio[ext=m4a]/best[ext=mp4]' \
  --merge-output-format mp4 \
  -o 'videos/%(playlist_index)02d_%(title)s.%(ext)s' \
  'https://www.youtube.com/playlist?list=PLZHQObOWTQDMsr9K-rj53DwVRMYO3t5Yr'
```

## Project Structure

```
3b1b-calculus/
├── _quarto.yml
├── index.qmd              # English index
├── index-zh.qmd           # Chinese index
├── lessons.qmd            # Auto-listing page
├── styles.css             # Manrope font + Desmos + Pyodide styling
├── CNAME                  # (to be configured later)
├── .gitignore             # Exclude /docs/, /.quarto/, /audio/, /videos/
├── .github/workflows/publish.yml  # GitHub Actions deployment
├── videos/                # Downloaded MP4s (gitignored)
├── audio/                 # 16kHz mono WAV (gitignored)
├── transcripts/           # ASR output .txt files
├── images/                # Key frames per lesson
│   ├── ch01/
│   ├── ch02/
│   └── ...
├── lessons/               # English lesson pages
│   ├── ch01-essence-of-calculus.qmd
│   ├── ch02-paradox-of-the-derivative.qmd
│   └── ...
└── lessons-zh/            # Chinese translated pages
    ├── ch01-essence-of-calculus.qmd
    └── ...
```

## Step-by-step Pipeline

### 1. Initialize Quarto Project

Create `_quarto.yml`:
```yaml
project:
  type: website
  output-dir: docs

website:
  title: "Essence of Calculus — 3Blue1Brown"
  navbar:
    left:
      - href: index.qmd
        text: Home
      - href: index-zh.qmd
        text: 中文版
      - href: lessons.qmd
        text: Lessons

format:
  html:
    theme: cosmo
    css: styles.css
    toc: true
```

### 2. Create styles.css

```css
@import url('https://fonts.googleapis.com/css2?family=Manrope:wght@300;400;500;600;700;800&display=swap');

body {
  font-family: 'Manrope', sans-serif !important;
}

.navbar-brand, .nav-link, h1, h2, h3, h4, h5, h6 {
  font-family: 'Manrope', sans-serif !important;
}

.desmos-container {
  width: 100%;
  height: 500px;
  margin: 1.5em 0;
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
}

.pyodide-container {
  width: 100%;
  margin: 1.5em 0;
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
  background: #1e1e2e;
}

.pyodide-container .code-input {
  width: 100%;
  min-height: 150px;
  padding: 1em;
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
  font-size: 14px;
  background: #1e1e2e;
  color: #cdd6f4;
  border: none;
  resize: vertical;
}

.pyodide-container .run-btn {
  padding: 0.5em 1.5em;
  margin: 0.5em;
  background: #89b4fa;
  color: #1e1e2e;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-family: 'Manrope', sans-serif;
  font-weight: 600;
}

.pyodide-container .output {
  padding: 1em;
  background: #181825;
  color: #a6e3a1;
  font-family: monospace;
  min-height: 50px;
  border-top: 1px solid #313244;
}

.pyodide-container .plot-output {
  background: white;
  text-align: center;
  padding: 1em;
}

.key-formula {
  background: #f8f9fa;
  border-left: 4px solid #0d6efd;
  padding: 1em 1.5em;
  margin: 1em 0;
  border-radius: 0 8px 8px 0;
}
```

### 3. Audio Conversion

For each video, convert to 16kHz mono WAV:
```bash
mkdir -p audio
for f in videos/*.mp4; do
  name=$(basename "$f" .mp4)
  ffmpeg -y -i "$f" -ar 16000 -ac 1 -f wav "audio/${name}.wav"
done
```

### 4. ASR Transcription

Use the local ASR server at `http://localhost:8080/v1/audio/transcriptions` with model `qwen3-asr`. Use the transcribe.py script pattern:

```python
#!/usr/bin/env python3
"""Transcribe lecture audio using local ASR endpoint."""
import base64, json, subprocess, tempfile, requests
from pathlib import Path

API = "http://localhost:8080/v1/audio/transcriptions"
AUDIO_DIR = Path("audio")
OUTPUT_DIR = Path("transcripts")
OUTPUT_DIR.mkdir(exist_ok=True)
CHUNK_SECS = 240  # 4 minutes per chunk

def get_duration(wav_path):
    result = subprocess.run(
        ["ffprobe", "-v", "quiet", "-show_entries", "format=duration",
         "-of", "csv=p=0", str(wav_path)],
        capture_output=True, text=True)
    return float(result.stdout.strip())

def transcribe_chunk(wav_path, language="en"):
    with open(wav_path, "rb") as f:
        audio_b64 = base64.b64encode(f.read()).decode("utf-8")
    resp = requests.post(API, json={
        "file": audio_b64, "model": "qwen3-asr", "language": language,
    }, timeout=300)
    if resp.status_code != 200:
        print(f"  ERROR {resp.status_code}: {resp.text[:200]}")
        return ""
    return resp.json().get("text", "")

audio_files = sorted(AUDIO_DIR.glob("*.wav"))
print(f"Found {len(audio_files)} audio files\n")

for af in audio_files:
    out_path = OUTPUT_DIR / f"{af.stem}.txt"
    if out_path.exists():
        print(f"Skip (exists): {af.name}")
        continue
    duration = get_duration(af)
    num_chunks = max(1, int(duration // CHUNK_SECS) + (1 if duration % CHUNK_SECS > 0 else 0))
    print(f"Transcribing: {af.name} ({duration:.0f}s, {num_chunks} chunks)")
    all_text = []
    for i in range(num_chunks):
        start = i * CHUNK_SECS
        chunk_dur = min(CHUNK_SECS, duration - start)
        with tempfile.NamedTemporaryFile(suffix=".wav", delete=True) as tmp:
            subprocess.run([
                "ffmpeg", "-y", "-i", str(af),
                "-ss", str(start), "-t", str(chunk_dur),
                "-acodec", "pcm_s16le", "-ar", "16000", "-ac", "1",
                tmp.name], capture_output=True, check=True)
            print(f"  Chunk {i+1}/{num_chunks} [{start:.0f}s-{start+chunk_dur:.0f}s]...")
            text = transcribe_chunk(tmp.name, language="en")
        if text:
            all_text.append(text)
    full_text = "\n".join(all_text)
    out_path.write_text(full_text)
    print(f"  -> Saved to {out_path}\n")
print("All done!")
```

### 5. Key Frame Extraction

Extract 4 key frames per video at 15%, 38%, 62%, 85% of duration. Resize to 800px width:
```bash
mkdir -p images
for f in videos/*.mp4; do
  name=$(basename "$f" .mp4)
  mkdir -p "images/${name}"
  dur=$(ffprobe -v quiet -show_entries format=duration -of csv=p=0 "$f")
  for i in 1 2 3 4; do
    case $i in
      1) ts=$(echo "$dur * 0.15" | bc);;
      2) ts=$(echo "$dur * 0.38" | bc);;
      3) ts=$(echo "$dur * 0.62" | bc);;
      4) ts=$(echo "$dur * 0.85" | bc);;
    esac
    ffmpeg -y -ss "$ts" -i "$f" -frames:v 1 -q:v 2 "/tmp/${name}_frame${i}.jpg"
    sips --resampleWidth 800 "/tmp/${name}_frame${i}.jpg" --out "images/${name}/frame_0${i}.jpg"
  done
done
```

### 6. GitHub Setup

Create repo `ymote/3b1b-calculus`, create v1.0 release, upload videos:
```bash
gh repo create ymote/3b1b-calculus --public --source=. --push
gh release create v1.0 --repo ymote/3b1b-calculus --title "Lecture Videos" --notes "3Blue1Brown Essence of Calculus video series"
for f in videos/*.mp4; do
  gh release upload v1.0 "$f" --repo ymote/3b1b-calculus
done
```

### 7. GitHub Actions Workflow

Create `.github/workflows/publish.yml`:
```yaml
name: Deploy Quarto site
on:
  push:
    branches: [main]
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: quarto-dev/quarto-actions/setup@v2
      - run: quarto render
      - uses: actions/upload-pages-artifact@v3
        with:
          path: docs
      - uses: actions/deploy-pages@v4
```

### 8. Lesson Page Format

Each lesson should follow this FORMAL, ACADEMIC style. Use college-level mathematical prose:

```markdown
---
title: "The Essence of Calculus"
date: 2023-04-01
---

This lesson develops the foundational intuition behind calculus through a single
geometric question: why is the area of a circle $\pi r^2$? Through this lens,
we introduce the three central ideas of calculus — integrals, derivatives, and
the fundamental theorem connecting them.

::: {.callout-note collapse="true"}
## Prerequisites

- Familiarity with the area formulas for basic geometric shapes (rectangles, triangles)
- Understanding of the concept of a function $f(x)$
- Comfort with the notation $\sum$ for finite sums
:::

## Topics Covered

- Approximating circular area via concentric ring decomposition
- The integral as the limit of Riemann sums
- The derivative as instantaneous rate of change
- The Fundamental Theorem of Calculus: integration and differentiation as inverse operations

## Lecture Video

```{=html}
<video controls width="100%" preload="metadata">
  <source src="https://github.com/ymote/3b1b-calculus/releases/download/v1.0/FILENAME.mp4" type="video/mp4">
</video>
```

## Key Video Frames

![](../images/ch01/frame_01.jpg)
![](../images/ch01/frame_02.jpg)
![](../images/ch01/frame_03.jpg)
![](../images/ch01/frame_04.jpg)

## Key Concepts

### Approximating the Area of a Circle

[Detailed formal mathematical content derived from transcript...]

### Interactive Desmos Graph

```{=html}
<div id="calc1" class="desmos-container"></div>
<script src="https://www.desmos.com/api/v1.9/calculator.js?apiKey=dcb31709b452b1cf9dc26972add0fda6"></script>
<script>
  var calc1 = Desmos.GraphingCalculator(document.getElementById('calc1'), {
    expressions: true, settingsMenu: false
  });
  // Add expressions relevant to the lesson
</script>
```

### Interactive Python (Pyodide)

```{=html}
<div class="pyodide-container">
  <textarea class="code-input" id="code1">
import numpy as np
import matplotlib.pyplot as plt

# Approximate area of circle using rings
N = 50
r_max = 3
dr = r_max / N
total_area = 0
rings_r = []
rings_area = []

for i in range(N):
    r = i * dr
    ring_area = 2 * np.pi * r * dr
    total_area += ring_area
    rings_r.append(r)
    rings_area.append(ring_area)

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(10, 4))
ax1.bar(rings_r, rings_area, width=dr*0.9, alpha=0.7, color='steelblue')
ax1.set_xlabel('Radius r')
ax1.set_ylabel('Ring area 2πr·dr')
ax1.set_title(f'Ring decomposition (N={N})')

ax2.bar(rings_r, np.cumsum(rings_area), width=dr*0.9, alpha=0.7, color='coral')
ax2.axhline(y=np.pi*r_max**2, color='black', linestyle='--', label=f'πr² = {np.pi*r_max**2:.2f}')
ax2.set_xlabel('Radius r')
ax2.set_ylabel('Cumulative area')
ax2.set_title(f'Cumulative sum → πr² = {np.pi*r_max**2:.4f}')
ax2.legend()

plt.tight_layout()
plt.savefig('/dev/stdout', format='svg')
  </textarea>
  <button class="run-btn" onclick="runPyodide('code1', 'output1', 'plot1')">Run ▶</button>
  <div class="output" id="output1"></div>
  <div class="plot-output" id="plot1"></div>
</div>
```

## Summary

::: {.key-formula}
| Concept | Key Result |
|---|---|
| Area of circle | $A = \int_0^R 2\pi r\, dr = \pi R^2$ |
| Derivative | $\frac{dA}{dR} = 2\pi R$ (the circumference) |
| Fundamental Theorem | $\frac{d}{dx}\int_0^x f(t)\,dt = f(x)$ |
:::
```

### 9. Pyodide Integration

Add this shared Pyodide loader script. Create `pyodide-loader.js`:

```javascript
let pyodideReady = null;

async function loadPyodideAndPackages() {
  if (pyodideReady) return pyodideReady;
  pyodideReady = (async () => {
    let pyodide = await loadPyodide();
    await pyodide.loadPackage(["numpy", "matplotlib"]);
    return pyodide;
  })();
  return pyodideReady;
}

async function runPyodide(codeId, outputId, plotId) {
  const codeEl = document.getElementById(codeId);
  const outputEl = document.getElementById(outputId);
  const plotEl = document.getElementById(plotId);

  outputEl.textContent = "Loading Python environment...";
  plotEl.innerHTML = "";

  try {
    const pyodide = await loadPyodideAndPackages();

    // Redirect stdout
    pyodide.runPython(`
import sys, io
sys.stdout = io.StringIO()
    `);

    // Set up matplotlib for SVG output
    pyodide.runPython(`
import matplotlib
matplotlib.use('AGG')
import matplotlib.pyplot as plt
import io, base64
plt.close('all')
    `);

    const code = codeEl.value;
    pyodide.runPython(code);

    // Capture stdout
    const stdout = pyodide.runPython("sys.stdout.getvalue()");
    outputEl.textContent = stdout || "(no text output)";

    // Capture plot if any
    const plotData = pyodide.runPython(`
import base64
buf = io.BytesIO()
if plt.get_fignums():
    plt.savefig(buf, format='svg', bbox_inches='tight')
    buf.seek(0)
    base64.b64encode(buf.read()).decode('utf-8')
else:
    ''
    `);

    if (plotData) {
      const svgData = atob(plotData);
      plotEl.innerHTML = svgData;
    }

    pyodide.runPython("sys.stdout = sys.__stdout__");

  } catch (err) {
    outputEl.textContent = "Error: " + err.message;
  }
}
```

Include in `_quarto.yml` under format:
```yaml
format:
  html:
    theme: cosmo
    css: styles.css
    toc: true
    include-in-header:
      text: |
        <script src="https://cdn.jsdelivr.net/pyodide/v0.25.1/full/pyodide.js"></script>
    include-after-body:
      file: pyodide-loader.js
```

### 10. Writing Style Guide (CRITICAL)

Use FORMAL ACADEMIC TONE throughout:
- "we" instead of "you"; "one may observe" instead of "notice that"
- "substitute" not "plug in"; "neglect" not "throw away"
- Section headers: "Prerequisites" not "What You Need to Know First"
- "Applications" or "Motivation" not "Real-World Connection"
- No exclamation marks in prose
- No casual hooks ("Have you ever wondered...")
- Use proper theorem/definition formatting
- Include rigorous mathematical derivations from the transcript
- Each lesson should have 2-3 interactive Desmos graphs AND 1-2 Pyodide code cells

### 11. Translation Conventions (Chinese Pages)

Create Chinese translations in `lessons-zh/` with these conventions:
- Key Ideas → 核心要点
- Applications/Motivation → 应用背景
- Summary → 速查表
- Example → 示例
- Lecture Video → 课程视频
- Key Video Frames → 课程关键帧
- Topics Covered → 本课内容
- Prerequisites → 预备知识
- Interactive demonstration → 交互演示
- Use formal Chinese: "我们" not "你"; no "!" exclamation marks
- Keep ALL HTML/Desmos/Pyodide/LaTeX blocks exactly as-is

### 12. Index Pages

`index.qmd` should list all chapters in a table:
```markdown
| Chapter | Topic |
|---------|-------|
| 1 | [The Essence of Calculus](lessons/ch01-essence-of-calculus.qmd) |
| 2 | [The Paradox of the Derivative](lessons/ch02-paradox-of-derivative.qmd) |
...
```

### 13. .gitignore

```
/.quarto/
/docs/
/audio/
/videos/
```

## Execution Order

1. Create project structure (_quarto.yml, styles.css, .gitignore, pyodide-loader.js, GitHub Actions workflow)
2. Download all playlist videos with yt-dlp
3. Convert to audio (16kHz mono WAV)
4. Run ASR transcription
5. Extract key frames (4 per video, 800px)
6. Create GitHub repo and upload videos to release
7. Generate lesson pages (EN) — read transcript + view frames → write formal content with Desmos + Pyodide
8. Generate lesson pages (CN) — translate EN pages
9. Create index pages (EN + CN)
10. Commit and push
11. Enable GitHub Pages
12. Verify deployment

## Important Notes

- The Desmos API key is: dcb31709b452b1cf9dc26972add0fda6
- The ASR server runs at http://localhost:8080/v1/audio/transcriptions with model qwen3-asr
- Videos go to GitHub Releases (v1.0), not the repo itself
- Use `sips --resampleWidth 800` (macOS) for image resizing
- GitHub repo owner is `ymote`
- Use Manrope Google Font for all text
- Platform is macOS (darwin)
