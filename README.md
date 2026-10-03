# mssg-plugins

mssg 的插件仓库。目前主要是 **Hugo 兼容 shortcode**：把对应 `.html` 丢进站点的 `templates/shortcodes/` 目录，文章里就能用 `{{< name >}}`。

## 插件列表

| 插件 | 说明 | Hugo 兼容 |
|------|------|-----------|
| [gist](shortcodes/gist/) | GitHub Gist 嵌入 | `{{< gist >}}` |
| [tweet](shortcodes/tweet/) | Twitter/X 推文嵌入 | `{{< tweet >}}` |
| [vimeo](shortcodes/vimeo/) | Vimeo 视频嵌入 | `{{< vimeo >}}` |
| [instagram](shortcodes/instagram/) | Instagram 帖子嵌入 | `{{< instagram >}}` |

## AI 插件

| 插件 | 说明 |
|------|------|
| [ai-writer](ai/ai-writer/) | AI 写作助手：续写、润色、扩写、起标题、写摘要 |

## 安装

手动：复制 `shortcodes/<name>/<name>.html` 到你站点的 `templates/shortcodes/` 下。

`plugins.json` 是给以后 App 内一键安装用的机器索引。

## 写插件

shortcode 插件就是一个 Jinja2 模板，可用变量：

- `args`：位置参数列表，如 `{{< gist spf13 7896402 >}}` → `args = ["spf13", "7896402"]`
- `kwargs`：键值参数字典，如 `{{< vimeo id="123" >}}` → `kwargs = {"id": "123"}`
- `page` / `site`：当前页面与站点配置

## 暂不支持的 Hugo shortcode

- `highlight`：需要配对标签（`{{< highlight >}}...{{< /highlight >}}`），mssg 解析器目前只支持自闭合；代码高亮直接用 Markdown 的 ``` 围栏代码块即可（Pygments）。
- `ref` / `relref`：需要页面索引，纯模板做不了，等 core 支持。
