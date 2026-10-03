# mssg Plugin Development Guide

mssg is a self-developed Python static site generator; 织网 ("ZhiWang") is its Android app.
Plugin repository: <https://github.com/fghijkln/mssg-plugins>

## 1. What plugins can do

| | shortcode plugin | ai plugin |
|---|---|---|
| What it is | A single Jinja2 template file | A JSON file defining a set of AI actions |
| Problem solved | Embedding third-party content in articles (videos, tweets, widgets…) | Adding buttons to the app's AI assistant (continue writing, fix bugs…) |
| Location in repo | `shortcodes/<name>/<name>.html` | `ai/<name>/<name>.json` |
| Installed to | `templates/shortcodes/` of a site | `files/ai_plugins/` in the app |
| How to use | Write `{{< name args >}}` in an article | App → editor → 🤖 → tap an action button |
| Needs APK changes? | Never | No since v0.26.0 (PluginHost, see §2) |
| Existing examples | gist, tweet, vimeo, instagram | ai-writer, ai-coder |

## 2. Manifest and plugin index

### 2.1 `plugins.json` (machine index at the repo root)

The app's extension store fetches this file, then downloads and installs each entry. Fields per entry:

```json
{
  "name": "bilibili",
  "type": "shortcode",
  "version": "1.0.0",
  "description": "Embed Bilibili videos",
  "files": [
    "shortcodes/bilibili/bilibili.html"
  ],
  "install_to": "templates/shortcodes/bilibili.html"
}
```

- `name`: plugin name. For shortcodes it is the `{{< name >}}` tag name; for ai plugins it becomes the installed file name. Stick to lowercase letters, digits and hyphens (see §6.1).
- `type`: currently only `shortcode` or `ai`.
- `version`: semantic version `x.y.z`.
- `description`: one-line description, shown in the extension store.
- `files`: files to download from the repo, relative to the repo root.
- `install_to`: install destination, relative to the target root.

### 2.2 Plugin manifest: `contributions` declarations (v0.26.0+)

Since v0.26.0 the APK is a generic plugin host (PluginHost): **it does not know plugin "types" — it only honors `contributions` declarations in the manifest.**
A plugin declares "I contribute these things to extension point X", and the host renders them there. New plugin capabilities only need a new extension-point contribution — no more APK patches.

Extension points available today:

| Extension point | Contribution format | Rendered at |
|---|---|---|
| `ai.actions` | `[{id, label, system, prompt}]` | Action buttons in the app's AI assistant |

New-format manifest example:

```json
{
  "name": "ai-translate",
  "version": "1.0.0",
  "description": "AI translation assistant",
  "contributions": {
    "ai.actions": [
      {
        "id": "zh2en",
        "label": "🌐 ZH→EN",
        "system": "You are a Chinese-English translator. Output only the translation, no explanations.",
        "prompt": "Translate the following Chinese text into English. Output only the translation:\n\n{{content}}"
      }
    ]
  }
}
```

### 2.3 Legacy format compatibility

ai-writer and ai-coder still use the legacy format (top-level `"type": "ai"` + `actions` array):

```json
{
  "name": "ai-writer",
  "type": "ai",
  "version": "1.0.0",
  "description": "AI writing assistant: continue, polish, expand, title, summarize",
  "actions": [
    {
      "id": "continue",
      "label": "✍️ Continue",
      "system": "You are a writing assistant. Output only the continuation, no explanations.",
      "prompt": "Continue the following article, keeping its language style and tone. Output only the continuation:\n\n{{content}}"
    }
  ]
}
```

PluginHost automatically normalizes the legacy format into `contributions["ai.actions"]`, so old plugins keep working with zero changes.
New plugins should use the `contributions` format directly.

The four action fields:

- `id`: unique action id within the plugin; use letters/digits/underscores.
- `label`: button text; emoji are welcome (easier to tap on a phone).
- `system`: system prompt defining the AI's role and output discipline.
- `prompt`: user prompt template — **`{{content}}` is replaced with the editor's body text**.

## 3. Hands-on: write a shortcode plugin

Goal: embed Bilibili videos with `{{< bilibili BV1xx411c7mD >}}`.

### Step 1: create the directory and template

```
shortcodes/bilibili/
  bilibili.html
  README.md
```

`bilibili.html` (complete and runnable, Jinja2, same style as the existing vimeo.html):

```html
{#- Bilibili video embed
    Usage: {{< bilibili BVID >}} or {{< bilibili bvid="BVID" >}}
    Available variables: args (positional args), kwargs (key=value args), page, site
-#}
{%- set bvid = kwargs.get("bvid") or (args[0] if args else "") -%}
{%- if bvid -%}
<div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
  <iframe src="https://player.bilibili.com/player.html?bvid={{ bvid }}" style="position:absolute;top:0;left:0;width:100%;height:100%;border:0;" allowfullscreen title="bilibili video"></iframe>
</div>
{%- endif -%}
```

Notes:

- Document usage in a `{#- ... -#}` header comment (all existing plugins do this).
- Accept both calling conventions: `kwargs.get("bvid") or (args[0] if args else "")`.
- Render nothing when the parameter is empty (`{%- if bvid -%}`) — never emit a broken iframe.

Available template variables (per the repo README): `args`, `kwargs`, `page`, `site`.
E.g. `{{< gist spf13 7896402 >}}` → `args = ["spf13", "7896402"]`;
`{{< vimeo id="123" >}}` → `kwargs = {"id": "123"}`.

### Step 2: test locally

1. Copy `bilibili.html` into your site's `templates/shortcodes/`.
2. Write `{{< bilibili BV1xx411c7mD >}}` in an article.
3. Run `mssg build` and open the generated HTML to confirm the iframe is embedded.
4. Also test bad input: `{{< bilibili >}}` (no args) should render nothing and not break the page.

### Step 3: write README.md

Follow the format of `shortcodes/gist/README.md`: plugin name, Hugo-compatibility note, install steps, and point usage docs at the template header comment.

### Step 4: register in `plugins.json`

Append to the `plugins` array:

```json
{
  "name": "bilibili",
  "type": "shortcode",
  "version": "1.0.0",
  "description": "Embed Bilibili videos (see template header for usage)",
  "files": [
    "shortcodes/bilibili/bilibili.html"
  ],
  "install_to": "templates/shortcodes/bilibili.html"
}
```

## 4. Hands-on: write an ai plugin

Goal: a translation assistant with 3 actions. File `ai/ai-translate/ai-translate.json` (complete and runnable):

```json
{
  "name": "ai-translate",
  "version": "1.0.0",
  "description": "AI translation assistant: ZH→EN, EN→ZH, polish English",
  "contributions": {
    "ai.actions": [
      {
        "id": "zh2en",
        "label": "🌐 ZH→EN",
        "system": "You are a Chinese-English translator. Output only the translation, no explanations.",
        "prompt": "Translate the following Chinese text into English. Output only the translation:\n\n{{content}}"
      },
      {
        "id": "en2zh",
        "label": "🌐 EN→ZH",
        "system": "You are an English-Chinese translator. Output only the translation, no explanations.",
        "prompt": "Translate the following English text into Chinese. Output only the translation:\n\n{{content}}"
      },
      {
        "id": "polish-en",
        "label": "✨ Polish English",
        "system": "You are an English editor. Output only the polished text, no explanations.",
        "prompt": "Polish the following English: fix grammar and make it sound native. Output only the result:\n\n{{content}}"
      }
    ]
  }
}
```

Notes:

- Every `prompt` must contain `{{content}}` — the app replaces it with the editor text. Without it, the button sends nothing to the AI.
- Be specific in `system`: "Output only the translation, no explanations" saves a lot of noise.
- Emoji in `label` make phone buttons easier to identify.
- `id` must be unique within the plugin.

### Test locally

1. Validate the JSON first: `python3 -m json.tool ai/ai-translate/ai-translate.json`.
2. ai plugins can currently only be tested end-to-end in the app: install via the extension store → open the editor → 🤖 → pick the plugin from the dropdown → tap an action and check the AI response.
   (TBD: whether a desktop test harness will be provided.)

## 5. Submit to the plugin repository

1. Fork `fghijkln/mssg-plugins` (or push a branch directly if you have write access).
2. Follow the directory conventions: `shortcodes/<name>/` or `ai/<name>/`, each with at least the main file + `README.md`.
3. Append an entry to the `plugins` array in `plugins.json` (fields, see §2.1).
4. Validate the index locally: `python3 -c "import json; json.load(open('plugins.json'))"`.
5. Open a PR describing what the plugin does and how you tested it.

## 6. Common pitfalls

1. **The plugin name is the security boundary.** The app whitelists plugin names against `../` path traversal, and `install_to` must stay inside whitelisted directories. Lowercase letters, digits and hyphens only — consistent with existing plugins is safest.
2. **Shortcodes must be self-closing.** `{{< name args >}}` works; paired tags like `{{< name }}...{{< /name >}}` are not supported (the mssg parser only recognizes self-closing tags).
3. **Unknown shortcode names are left as-is.** If you see a literal `{{< xxx >}}` in the rendered page, the name is probably misspelled — that's a debugging hint, not a bug.
4. **kwargs values are always strings.** In `{{< vimeo id="123" >}}`, `id` is the string `"123"` — don't do arithmetic on it in templates.
5. **An ai plugin is only "button definitions".** Actual AI calls go through the user's own configured endpoint (OpenAI-compatible) in the app; the plugin stores no keys and makes no network calls. Don't assume which model is behind it when writing prompts.
6. **Don't misspell `{{content}}`.** Without it, tapping the action sends nothing to the AI.
7. **Version semantics**: bump z for bug fixes, y for new actions/parameters, x for schema changes.
8. **Don't skip the README.** Every plugin directory needs a `README.md` with usage docs — you won't remember the parameter names in six months either.
