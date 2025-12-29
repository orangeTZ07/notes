### 常用vscode+vim插件快捷键组合

- ***适用于linux,部分适用于windows***
> 本内容将尽可能减少对快捷键的修改
```
Extension (VsCodeVim) version: 1.32.4
VSCode version: 1.107.1
OS: Arch Linux
```

> **tips** vim的大部分快捷键不再介绍,并且大部分快捷键和命令可用
> **notice** 很多vscode的快捷键会被vim插件捕获，从而无法使用
-------
#### 命令行
- 以下操作就用命令行完成把
  - 修改文件名
  - 移动文件
  - git操作（也可以用vscode快捷键）
  - c/cpp编译执行（不用配置了，好也（雾
-------
#### AI交互
- 修改vscode的AI内联聊天快捷键为ctrl+shift+i,避免被vim插件霸占从而导致无法使用
  - ***好像很多快捷键加个shift或者tab之类的没被vim使用的控制键就能避免被vim插件捕捉（？***
  
#### 终端操作
- ctrl + `显示终端
  - 这个快捷键经常可以用来快速回到编辑区（焦点在别处时按两次ctrl+`就可以把焦点换回编辑区
- ctrl + shift + `新建终端
- exit命令来关闭终端（关闭终端快捷键被霸占了）
- ctrl + tab快速切换打开的标签页(带上shift可以反向遍历打开的标签页)
- ctrl + PAGEUP/PAGEDOWN切换终端(numlock关了，9就是PAGEUP,3是PAGEDOWN)
- 关闭终端用exit命令把

#### 打开文件
- ctrl + shift + e打开vs内置资源管理，平时开文件主要靠这个
  - 在资源管理页面，左方向可以折叠文件夹，右方向可以展开文件夹
  - 选中文件后enter可以打开文件
- ctrl + p可以快速打开文件
  - 要使用该快捷键，可以先ctrl+`把焦点切换到终端，然后用ctrl+p
- ctrl + shift + t打开刚刚关闭的标签页
  
#### vscode交互
- ctrl + shift + p打开vscode快捷命令
- ctrl + , 打开设置
- alt + [对应字母] 可以点开含有字母提示的按钮，比如`alt+F`打开左上角的`文件`

#### 注释
- 可以修改一下单行注释快捷键
  - `ctrl + /` 改成 `ctrl + shift + /`
- ctrl+shift+A多行注释
- vim按v选中行之后按注释快捷键可以快速注释

#### 快速移动行
- alt + [上方向] 当前行与其前边一行换位置
- alt + [下方向] 当前行与其后边一行换位置

#### 缩进
- ctrl + [ 所选行删掉一层缩进 改为 `ctrl + shift + -`
- ctrl + ] 所选行删掉一层缩进 改为 `ctrl + shift + =`