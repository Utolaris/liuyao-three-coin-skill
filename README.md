# Liuyao Three Coin Skill

Codex skill for three-coin liuyao workflows with strict physical-order handling.

The core rule is simple: input records are treated as physical generation order, bottom-up from 初爻 to 上爻, while output diagrams are presented top-down for reading. This avoids the common mistake of reversing a six-line cast just because the final hexagram is displayed from top to bottom.

## What It Does

- Parses `line_values`, `tosses`, or natural-language `raw_text`.
- Defaults all ordered inputs to bottom-up physical order.
- Only accepts top-down input when `label_order: "top-down"` is explicitly present in the structured payload.
- Uses the fixed three-coin mapping:
  - `背` / `光面` / `字` = yang = `3`
  - `符` / `花面` / `面` = yin = `2`
- Returns top-down presentation fields such as `diagram_top_down`, `lines_top_down`, and `transformed_lines_top_down`.
- Blocks sensitive zi-hour cases when true-solar location data is missing.

## Install As A Codex Skill

Install from GitHub:

```bash
codex skills install github:Utolaris/liuyao-three-coin-skill
```

If your Codex build uses a local skills folder, you can also clone it directly:

```bash
git clone https://github.com/Utolaris/liuyao-three-coin-skill.git ~/.codex/skills/liuyao-three-coin
```

## Basic Input

```json
{
  "question": "这个项目下个月能不能落地？",
  "tosses": [
    "1: 3符",
    "2: 3背",
    "3: 2符1背",
    "4: 3符",
    "5: 3符",
    "6: 2背1符"
  ]
}
```

The first toss is interpreted as 初爻. The sixth toss is interpreted as 上爻.

## Explicit Top-Down Input

Use this only when the six values are genuinely provided from 上爻 to 初爻:

```json
{
  "question": "这次合作能不能顺利推进？",
  "line_values": [8, 9, 7, 6, 8, 7],
  "label_order": "top-down"
}
```

Natural-language claims such as `我这次是从上往下记的` do not override the default. The override must be a structured `label_order` field.

## Local Smoke Test

Create a payload:

```bash
cat > /tmp/liuyao-payload.json <<'JSON'
{
  "question": "这个项目下个月能不能落地？",
  "tosses": ["1: 3符", "2: 3背", "3: 2符1背", "4: 3符", "5: 3符", "6: 2背1符"]
}
JSON
```

Run the embedded single-file engine from the repository root:

```bash
uv run python - "$PWD/SKILL.md" --input /tmp/liuyao-payload.json --pretty <<'PY'
import base64
import gzip
import hashlib
import re
import runpy
import sys
import tempfile
from pathlib import Path

ENGINE_SHA256 = "0223cc6189ea871d460051ddc316e188884bf852bd2afd4c92ac459489c8e37d"
VENDOR_SHA256 = "2dc3b2491e950e15ee975825b9070fbeefa9d69b46703824e79341da42784e6e"
BUNDLE_ID = "f667093c2faad1d2"

skill_path = Path(sys.argv[1]).resolve()
engine_args = sys.argv[2:]
cache_dir = Path(tempfile.gettempdir()) / "liuyao_three_coin_singlefile" / BUNDLE_ID
cache_dir.mkdir(parents=True, exist_ok=True)

engine_path = cache_dir / "liuyao_engine.py"
vendor_path = cache_dir / "lunar_python_vendor.zip"

def sha256_file(path: Path) -> str | None:
    if not path.exists():
        return None
    return hashlib.sha256(path.read_bytes()).hexdigest()

def read_blob(text: str, tag: str) -> str:
    m = re.search(rf"<!--\s*{tag}_BEGIN\s*(.*?)\s*{tag}_END\s*-->", text, re.S)
    if not m:
        raise SystemExit(f"missing embedded payload: {tag}")
    return re.sub(r"\s+", "", m.group(1))

if sha256_file(engine_path) != ENGINE_SHA256 or sha256_file(vendor_path) != VENDOR_SHA256:
    text = skill_path.read_text(encoding="utf-8")
    engine_blob = read_blob(text, "ENGINE_B64_GZ")
    vendor_blob = read_blob(text, "VENDOR_B64_GZ")
    engine_path.write_bytes(gzip.decompress(base64.b64decode(engine_blob)))
    vendor_path.write_bytes(gzip.decompress(base64.b64decode(vendor_blob)))

if sha256_file(engine_path) != ENGINE_SHA256:
    raise SystemExit("engine hash verification failed")
if sha256_file(vendor_path) != VENDOR_SHA256:
    raise SystemExit("vendor hash verification failed")

sys.argv = [str(engine_path), *engine_args]
runpy.run_path(str(engine_path), run_name="__main__")
PY
```

A successful run returns JSON with `status: "ok"` and an order warning confirming the bottom-up default.

## Safety Boundaries

- Missing or extra lines return `fatal_error`.
- Invalid three-coin totals such as `2符2背` return `fatal_error`.
- Unauthorized `coin_map` overrides return `fatal_error`.
- Zi-hour boundary casts without location assurance return `system_pause`.

