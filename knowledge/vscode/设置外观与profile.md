# 设置与外观：光标、工作区、字体、Profile

> 来源：本机 VS Code 1.143 实测记录。标 **待确认** 的是推断，未复核。

## 光标

改颜色（**已验证**）：

```jsonc
"workbench.colorCustomizations": {
  "editorCursor.foreground": "#ff0000"
}
```

`editor.cursorForeground` **不存在**，写进 settings 只会被标成未知设置。`editor.cursorWidth` 是像素值，2–3 才看得明显。

## 工作区设置不用手动建目录

设置界面切到"工作区"标签改任意值，VS Code 会**自动创建** `.vscode/settings.json`。

## 搜索搜不到某些词是正常的

设置搜索主要匹配**设置 ID 和标题**，只在描述里出现的词搜不到。典型例子：搜 "IntelliSense" 没有结果，因为它在描述里而不是标题里。

## 字体

`editor.fontFamily` + `editor.fontLigatures`（连字要字体本身支持）。字体没装时设置**照样生效但视觉无变化**——排查字体问题先确认字体真的装了。

## Profile

管整套隔离：扩展 + 设置 + 键位 + 代码片段，官方文档另列 **MCP servers、tasks、UI 布局**（已查证，2026-10 文档）。

**旧结论已更正**：曾记为"只能手动切换、不会按文件夹自动套用"。当前官方文档（code.visualstudio.com/docs/configure/profiles）明确支持**文件夹/工作区关联**——选中 Profile 时关联到当前文件夹，之后再打开该文件夹自动激活，关联列表在 Profiles 编辑器的 Folders & Workspaces。

当前真实限制（已查证）：

- **Profile 之间没有继承**，不能只覆盖某项设置而沿用另一个 Profile；新建 Profile 是一次性复制，不是链接（vscode#188612 仍未实现）。
- 用默认 Profile 时界面不显示名字。
- 导出会剔除**机器相关设置**（指向本地路径那类）。
- 扩展不经 Settings Sync 同步到远程窗口（SSH / dev container / WSL）。
- 临时 Profile 不跨会话保存，关闭即删。
