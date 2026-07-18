## Custom (self-hosted) wake word models - two gotchas

If you're hosting your own `micro_wake_word` model manifest (e.g. on a self-hosted Gitea/Forgejo/GitHub instance) instead of using the stock shorthand names, and the model compiles but never shows up as a selectable wake word in Home Assistant - or worse, you get a config error like:

```
Failed config
micro_wake_word: [source .../respeaker-satellite-base.yaml:1358]
  [micro_wake_word] is an invalid option for [<root>].
```

...it's very unlikely to be your YAML. Both symptoms below trace back to the model's `.json` manifest itself, and ESPHome's error reporting doesn't point at the real cause - the failure happens deep inside a final validation pass that fetches and parses every model's manifest, and it can't map an error inside the manifest JSON back onto a location in your YAML config tree.

### 1. Wrong URL - must be the *raw* file, not the web view

If you're linking to a manifest hosted in a Gitea repo, the URL must use the raw-content route:

```
https://<host>/<owner>/<repo>/raw/branch/<branch>/<path-to-file>.json
```

Not the repo browser page (`/src/branch/...`), which returns an HTML page instead of JSON, and not a bare path with no branch segment, which 404s. Both look plausible when copied from the address bar after clicking around the Gitea UI, but only `raw/branch/...` serves the actual file content.

### 2. Manifest must match ESPHome's schema exactly - no extra fields

ESPHome's `MANIFEST_SCHEMA_V2` (`esphome/components/micro_wake_word/__init__.py`) is strict - it does not allow unrecognized top-level keys. If your manifest was produced by a third-party trainer that adds its own metadata (e.g. `label`, `model_format`, `quantization`, `sample_rate`, or custom sections like `tater_native`/`calibration`), validation fails as soon as ESPHome actually fetches and parses it.

A valid v2 manifest should contain *only*:

```json
{
  "type": "micro",
  "wake_word": "your_wake_word",
  "author": "...",
  "website": "https://...",
  "model": "your_model.tflite",
  "trained_languages": ["en"],
  "version": 2,
  "micro": {
    "probability_cutoff": 0.45,
    "sliding_window_size": 3,
    "feature_step_size": 10,
    "tensor_arena_size": 30000,
    "minimum_esphome_version": "2024.7.0"
  }
}
```

Strip anything else out before publishing the manifest for ESPHome to consume, even if your training tool wants to keep richer metadata elsewhere.

### Debugging tip

If you hit a similarly opaque error from `micro_wake_word` and want to know the *real* cause rather than guess from the error text, install the exact ESPHome version you're building with into a throwaway virtualenv and call the schema directly against your fetched manifest:

```bash
pip install esphome==<your version>
python3 -c "
import json
from esphome.components.micro_wake_word import MANIFEST_SCHEMA_V2
import esphome.config_validation as cv

manifest = json.load(open('your_model.json'))
try:
    MANIFEST_SCHEMA_V2(manifest)
    print('VALID')
except cv.Invalid as e:
    print('INVALID:', e)
"
```

This gives you the actual voluptuous error (e.g. `extra keys not allowed @ data['label']`) instead of the generic root-level message ESPHome shows in `esphome validate` / `esphome run`.
