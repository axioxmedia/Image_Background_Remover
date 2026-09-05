<div align="center">
  <h1>BG Key Desk</h1>
  <p><sub>去背景台 · v1.1</sub></p>
  <h3>Drop any image, normalize it to PNG, then cut the background on your own machine.</h3>
  <p>Automatic cutout or chroma key. Forced color + strength from <code>0.001</code> to <code>1.0</code>.</p>
  <p>
    <a href="#install">Install</a> ·
    <a href="#whats-in-1-1">v1.1</a> ·
    <a href="#features">Features</a> ·
    <a href="#requirements">Requirements</a> ·
    <a href="#architecture">Architecture</a> ·
    <a href="#documentation">Docs</a>
  </p>
</div>

> [!NOTE]
> The Windows EXE is unsigned PyInstaller output. Defender may warn on first launch. That is expected for a local build.
>
> The first **Auto** cut downloads `u2net` into `~/.u2net`. Later runs reuse the file and, in the same session, the loaded ONNX session.

> [!WARNING]
> Do not commit `.venv`, `dist/`, `build/`, `work/`, `*.exe`, `bg_key_desk.log`, or `model_eta.json`. Those are local runtime artifacts.

## At a glance

| | BG Key Desk v1.1 |
|---|---|
| **Workflow** | Drop image → convert to PNG → Auto or chroma-key cut → Save As PNG |
| **Auto engine** | `rembg` + `u2net` when available; local edge-flood fallback if the model cannot load |
| **Chroma key** | Hex color or click-to-sample, strength default `0.001`, maximum `1.0` |
| **Progress** | Modal bar on convert / cut / save. Auto mode shows a machine-based ETA countdown |
| **Preview** | Contained in the right pane: scale down to fit, keep aspect, never stretch to full width |
| **Save** | Native Save As dialog; never overwrites the source file |
| **UI** | Dark glass, black / gold `#e7c07a`, 中文 + EN |
| **Runtime** | FastAPI on `127.0.0.1` + pywebview |
| **Brand** | Packed via Axiox Media |

<a id="whats-in-1-1"></a>

## What’s in v1.1

This is the current shipped source, not the first drop.

| Change | Behavior |
|---|---|
| Model ETA | Auto cut estimates load + infer time from CPU cores, RAM, frozen-EXE flag, whether `u2net` is already on disk, and whether the session is already warm |
| Live countdown | The progress modal shows remaining seconds and a kind label: first download / local load / infer |
| Learned timing | Actual duration is blended into `model_eta.json` next to the EXE (`55%` history + `45%` probe) |
| Session reuse | `rembg` `u2net` session stays in memory after the first Auto cut in this process |
| Preview fit | Large images no longer overflow the right pane. Display scale is `min(1, paneW / imgW, paneH / imgH)` |
| Width | Preview width is auto from that scale. It is not stretched to 100% of the pane |
| Height | Display height follows the image aspect after the scale is applied |
| Color pick | Click-to-sample still maps through the CSS display size back to source pixels |

<a id="install"></a>

## Install

### 1. GitHub Deploy Desk (recommended)

One-click deploy this repository with [GitHub Deploy Desk](https://github.com/axioxmedia/github-deployer).

1. Get the deployer: [axioxmedia/github-deployer](https://github.com/axioxmedia/github-deployer)
2. Paste this repo URL into Deploy Desk: `https://github.com/axioxmedia/Image_Background_Remover`
3. Read the README in the app, then confirm deploy.

That is the supported install path. Use the source / EXE steps below only if you already have a local checkout.

### 2. Run from source or freeze an EXE

| Platform | Package | Guide |
|---|---|---|
| Windows 10/11 | `dist\BgKeyDesk.exe` after a local build | [HOW_TO_BUILD.txt](HOW_TO_BUILD.txt) |
| Source | Python 3.11 or 3.12 | commands below |

```bat
python -m venv .venv
.venv\Scripts\python.exe -m pip install -r requirements.txt
.venv\Scripts\python.exe app.py
```

To freeze a windowed EXE:

1. Install Python 3.11 or 3.12 from python.org and check **Add python.exe to PATH**.
2. Double-click `build_exe.bat`, or run `powershell -ExecutionPolicy Bypass -File .\build_exe.ps1`.
3. Open `dist\BgKeyDesk.exe`.

Rebuild the EXE after every source change. An old `dist\*.exe` will not update itself.

<a id="features"></a>

## Features

| Feature | Detail |
|---|---|
| Drop any common raster | JPEG, PNG, WEBP, BMP, GIF first frame, TIFF |
| Normalize first | Every source becomes an 8-bit RGBA PNG before cutout |
| Auto remove | Default mode. Uses rembg when the model can load |
| Chroma key | Pick a color, or click the preview to sample it |
| Strength | Slider `0.001`–`1.0`. Higher values key a wider color range |
| Progress modal | Convert, cut, and save each show a popup bar |
| Model ETA | Dynamic seconds while the Auto model is loading |
| Contained preview | Fit inside the pane; no full-width stretch; no overflow |
| Save As | System dialog; suggested name `{stem}_nobg.png` |
| Language | 中文 / EN, remembered in `localStorage` |

<a id="requirements"></a>

## Requirements

| | Minimum | Notes |
|---|---|---|
| OS | Windows 10/11 for the EXE | Source also runs where Python + a desktop WebView exist |
| Python | 3.11 or 3.12 | Only needed to run from source or to build |
| Disk | ~2 GB for a frozen build plus the u2net model | First Auto run needs network for the model |
| RAM | 4 GB | 8 GB is more comfortable on large photos |

<a id="architecture"></a>

## Architecture

```
pywebview window
    → http://127.0.0.1:<free port from 8787>
        → FastAPI
            → POST /api/jobs/normalize
            → POST /api/jobs/{id}/remove
            → GET  /api/jobs/{id}/events   (SSE: percent, label, eta_seconds, eta_kind)
            → GET  /api/jobs/{id}/preview.png
            → GET  /api/jobs/{id}/result.png
            → POST /api/jobs/{id}/pick-save
            → POST /api/jobs/{id}/save
        → static/index.html + styles.css + app.js
PyInstaller onefile EXE (console=False)
```

Temporary job files live next to the EXE under `work/`. Runtime logs go to `bg_key_desk.log`. Learned model timings go to `model_eta.json`.

ETA probe uses `os.cpu_count()`, Windows `GlobalMemoryStatusEx` or `/proc/meminfo`, `~/.u2net/u2net.onnx` presence, and `sys.frozen`.

<a id="documentation"></a>

## Documentation

<details>
<summary>What does strength do?</summary>

Strength is a 0–1 RGB distance threshold. `0.001` only keys pixels almost identical to the chosen color. `1.0` keys the full RGB distance range. A small feather is applied around the threshold so edges are not a hard stencil.

</details>

<details>
<summary>Why is the first Auto cut slow?</summary>

`u2net` is about 176 MB. The first call may download it, then load it into ONNX Runtime. The modal countdown is an estimate for that machine, not a fixed advertisement. Later cuts in the same process skip the load.

</details>

<details>
<summary>Preview looks small</summary>

That is intended in v1.1. The image is scaled to fit the pane and is never upscaled past 1:1. It will not use the full pane width unless the photo is wide enough to need it.

</details>

<details>
<summary>Startup failed</summary>

Send `bg_key_desk.log` from the folder that contains `BgKeyDesk.exe`. Do not send a screenshot only.

</details>

<details>
<summary>Run the UI in a browser</summary>

```
python app.py --web
```

Save As then falls back to the browser download path if the native dialog is missing.

</details>
