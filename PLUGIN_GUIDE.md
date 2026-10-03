# mssg 插件开发教程

mssg 是自研 Python 静态站点生成器，织网是它的 Android App。
插件仓库：<https://github.com/fghijkln/mssg-plugins>

## 1. 插件能做什么

| | shortcode 插件 | ai 插件 |
|---|---|---|
| 是什么 | 一个 Jinja2 模板文件 | 一个 JSON 文件，定义一组 AI 动作 |
| 解决什么 | 文章里嵌入第三方内容（视频、推文、小组件…） | 给 App 的 AI 助手加按钮（续写、修 bug…） |
| 仓库位置 | `shortcodes/<名>/<名>.html` | `ai/<名>/<名>.json` |
| 装到哪里 | 站点的 `templates/shortcodes/` | App 的 `files/ai_plugins/` |
| 怎么用 | 文章里写 `{{< 名 参数 >}}` | App → 编辑器 → 🤖 → 点动作按钮 |
| 要改 APK 吗 | 从来不用 | v0.26.0 起不用（PluginHost，见第 2 节） |
| 现有例子 | gist、tweet、vimeo、instagram | ai-writer、ai-coder |

## 2. manifest 与插件索引

### 2.1 `plugins.json`（仓库根目录的机器索引）

App 的扩展商店从仓库拉取这个文件，按条目下载、安装。每个条目字段：

```json
{
  "name": "bilibili",
  "type": "shortcode",
  "version": "1.0.0",
  "description": "B 站视频嵌入",
  "files": [
    "shortcodes/bilibili/bilibili.html"
  ],
  "install_to": "templates/shortcodes/bilibili.html"
}
```

- `name`：插件名。shortcode 插件里它就是 `{{< name >}}` 的名字；ai 插件里它是安装后的文件名。建议只用小写字母、数字、连字符（见第 6 节第 1 条）。
- `type`：目前只有 `shortcode` 或 `ai`。
- `version`：语义版本 `x.y.z`。
- `description`：一句话说明，会显示在扩展商店里。
- `files`：仓库中需要下载的文件，相对仓库根目录。
- `install_to`：安装到目标位置的相对路径。

### 2.2 插件 manifest：`contributions` 声明（v0.26.0+）

v0.26.0 起 APK 是通用插件宿主（PluginHost）：**它不认识"插件类型"，只认 manifest 里的 `contributions` 声明**。
插件声明"我给某个扩展点贡献了这些东西"，宿主就渲染到哪里。以后新增插件能力，只要声明新的扩展点贡献，装上即用，不用再改 APK。

目前已有的扩展点：

| 扩展点 | 贡献内容格式 | 渲染位置 |
|---|---|---|
| `ai.actions` | `[{id, label, system, prompt}]` | App → AI 助手的动作按钮 |

新格式 manifest 示例：

```json
{
  "name": "ai-translate",
  "version": "1.0.0",
  "description": "AI 翻译助手",
  "contributions": {
    "ai.actions": [
      {
        "id": "zh2en",
        "label": "🌐 中译英",
        "system": "你是一个中英翻译，直接输出译文，不要解释。",
        "prompt": "把下面的中文翻译成英文，直接输出译文：\n\n{{content}}"
      }
    ]
  }
}
```

### 2.3 老格式兼容

ai-writer、ai-coder 用的还是老格式（顶层 `type: "ai"` + `actions` 数组）：

```json
{
  "name": "ai-writer",
  "type": "ai",
  "version": "1.0.0",
  "description": "AI 写作助手：续写、润色、扩写、起标题、写摘要",
  "actions": [
    {
      "id": "continue",
      "label": "✍️ 续写",
      "system": "你是一个写作助手，直接输出内容，不要解释。",
      "prompt": "请续写下面的文章，保持原有的语言风格和语气，直接输出续写的内容：\n\n{{content}}"
    }
  ]
}
```

PluginHost 会自动把老格式归一化为 `contributions["ai.actions"]`，老插件零改动继续工作。
新插件建议直接用 `contributions` 格式。

action 四个字段：

- `id`：动作唯一标识，插件内唯一，用英文字母/数字/下划线。
- `label`：按钮上显示的文字，支持 emoji（手机上更好认）。
- `system`：系统提示词，定义 AI 的角色和输出纪律。
- `prompt`：用户提示词模板，**`{{content}}` 会被替换为编辑器里的正文**。

## 3. 手把手：写一个 shortcode 插件

目标：B 站视频嵌入，文章里写 `{{< bilibili BV1xx411c7mD >}}` 就能播视频。

### 步骤 1：建目录和模板文件

```
shortcodes/bilibili/
  bilibili.html
  README.md
```

`bilibili.html`（完整可运行，Jinja2 语法，风格照着现有 vimeo.html）：

```html
{#- Bilibili 视频嵌入
    用法：{{< bilibili BV号 >}} 或 {{< bilibili bvid="BV号" >}}
    可用变量：args（位置参数列表）、kwargs（键值参数字典）、page、site
-#}
{%- set bvid = kwargs.get("bvid") or (args[0] if args else "") -%}
{%- if bvid -%}
<div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
  <iframe src="https://player.bilibili.com/player.html?bvid={{ bvid }}" style="position:absolute;top:0;left:0;width:100%;height:100%;border:0;" allowfullscreen title="bilibili video"></iframe>
</div>
{%- endif -%}
```

写法要点：

- 文件头用 `{#- ... -#}` 写清用法注释（现有插件都这样）。
- 参数同时支持位置和键值两种写法：`kwargs.get("bvid") or (args[0] if args else "")`。
- 参数为空时输出空字符串（`{%- if bvid -%}`），别输出坏掉的 iframe。

模板里可用的变量（见仓库 README）：`args`、`kwargs`、`page`、`site`。
例如 `{{< gist spf13 7896402 >}}` → `args = ["spf13", "7896402"]`；
`{{< vimeo id="123" >}}` → `kwargs = {"id": "123"}`。

### 步骤 2：本地测试

1. 把 `bilibili.html` 拷到你站点的 `templates/shortcodes/` 下。
2. 在某篇文章里写 `{{< bilibili BV1xx411c7mD >}}`。
3. 跑 `mssg build`，打开生成的 HTML，确认 iframe 正常嵌入。
4. 异常输入也要测：`{{< bilibili >}}`（无参数）应该输出空，不破坏页面。

### 步骤 3：写 README.md

照 `shortcodes/gist/README.md` 的格式：插件名、Hugo 兼容说明、安装方法、用法指到模板头部注释。

### 步骤 4：注册到 `plugins.json`

在 `plugins` 数组末尾加：

```json
{
  "name": "bilibili",
  "type": "shortcode",
  "version": "1.0.0",
  "description": "B 站视频嵌入（用法见模板头部注释）",
  "files": [
    "shortcodes/bilibili/bilibili.html"
  ],
  "install_to": "templates/shortcodes/bilibili.html"
}
```

## 4. 手把手：写一个 ai 插件

目标：翻译助手，3 个动作。文件 `ai/ai-translate/ai-translate.json`（完整可运行）：

```json
{
  "name": "ai-translate",
  "version": "1.0.0",
  "description": "AI 翻译助手：中译英、英译中、润色译文",
  "contributions": {
    "ai.actions": [
      {
        "id": "zh2en",
        "label": "🌐 中译英",
        "system": "你是一个中英翻译，直接输出译文，不要解释。",
        "prompt": "把下面的中文翻译成英文，直接输出译文：\n\n{{content}}"
      },
      {
        "id": "en2zh",
        "label": "🌐 英译中",
        "system": "你是一个英中翻译，直接输出译文，不要解释。",
        "prompt": "把下面的英文翻译成中文，直接输出译文：\n\n{{content}}"
      },
      {
        "id": "polish-en",
        "label": "✨ 润色英文",
        "system": "你是一个英文编辑，直接输出润色后的英文，不要解释。",
        "prompt": "润色下面的英文，修正语法错误，让表达更地道，直接输出结果：\n\n{{content}}"
      }
    ]
  }
}
```

写法要点：

- `prompt` 里必须有 `{{content}}`——App 会把它替换成编辑器正文。没有它，点了按钮也没东西发给 AI。
- `system` 越具体越好，"直接输出译文，不要解释"这种指令能省掉一堆废话。
- `label` 加 emoji，手机按钮更好认。
- `id` 在插件内唯一。

### 本地测试

1. 先校验 JSON 格式：`python3 -m json.tool ai/ai-translate/ai-translate.json`。
2. ai 插件目前只能在 App 里实测：扩展商店安装 → 打开编辑器 → 🤖 → 下拉选插件 → 点动作按钮，看 AI 返回是否符合预期。
   （待确认：以后是否提供桌面端测试工具。）

## 5. 提交到插件仓库

1. Fork `fghijkln/mssg-plugins`（有写权限可直接 push 分支）。
2. 按目录规范放文件：`shortcodes/<名>/` 或 `ai/<名>/`，每个目录下至少有主文件 + `README.md`。
3. 在 `plugins.json` 的 `plugins` 数组末尾加条目（字段见 2.1 节）。
4. 本地校验索引合法：`python3 -c "import json; json.load(open('plugins.json'))"`。
5. 提 PR，说明插件做什么、怎么测的。

## 6. 常见坑

1. **插件名就是安全边界**。App 对插件名有白名单校验，防 `../` 路径穿越；`install_to` 也只能落在白名单目录下。名字只用小写字母、数字、连字符，跟现有插件保持一致最保险。
2. **shortcode 只支持自闭合**。`{{< name args >}}` 可以，`{{< name }}...{{< /name >}}` 这种配对标签目前不支持（mssg 解析器只认自闭合）。
3. **未知名字的 shortcode 会保留原文**。页面上直接看到 `{{< xxx >}}` 没被渲染，多半是名字拼错了——这是排查线索，不是 bug。
4. **kwargs 的值永远是字符串**。`{{< vimeo id="123" >}}` 里 `id` 是 `"123"`，模板里别当数字做运算。
5. **ai 插件只是"按钮定义"**。真正的 AI 调用走 App 里用户自己配的接口（OpenAI 兼容），插件不存 key、不调网络。写 prompt 时别假设用的是哪个模型。
6. **`{{content}}` 别拼错**。少了它，动作点了也没内容发给 AI。
7. **版本号语义**：修 bug 升 z，加动作/加参数升 y，改字段结构升 x。
8. **README 别偷懒**。每个插件目录都要有 `README.md` 写清用法——半年后你自己都忘了参数叫什么。
