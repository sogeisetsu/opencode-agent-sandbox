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

管整套隔离：扩展 + 设置 + 键位 + 代码片段。局限：**只能手动切换**，不会按文件夹自动套用。
