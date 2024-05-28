---
title: ssh下通过osc52使用nvim剪切板
tags:
  - linux
  - nvim
  - ssh
categories:
  - [linux, nvim]
  - [linux, ssh]
date: 2024-05-21 13:43:41
---


> 本文旨于记录作者利用`lazy.nvim`插件管理器安装`ojroques/nvim-osc52`到neovim的过程  
> 参考:<br>
> https://github.com/ojroques/vim-oscyank
> https://github.com/ojroques/nvim-osc52
> https://www.reddit.com/r/vim/comments/k1ydpn/a_guide_on_how_to_copy_text_from_anywhere/

没想到围绕一个剪切板转发能记录这么多次....
之前的做法是透过x11转发xclip剪切板到本地x服务器，这种做法我认为不太好。除了有[潜在的安全问题](https://askubuntu.com/questions/101829/why-is-x11-a-security-risk-in-servers)外，还有一个就是为了个剪切板搬来整个x11。无论怎么说都不是个优雅的方案<br>
`ojroques/vim-oscyank`和`ojroques/nvim-osc52`分别是vim和neovim下的插件，目的是在vim和neovim下实现osc52转义序列以将文本从终端复制到系统剪切板<br>
osc52是ansi下的转义序列，可以实现将文本从终端复制到系统剪切板而不考虑其他东西，详见[A guide on how to copy text from anywhere, including through SSH, with OSC52 ](https://www.reddit.com/r/vim/comments/k1ydpn/a_guide_on_how_to_copy_text_from_anywhere/)
> [neovim 10.0版本](https://github.com/neovim/neovim/pull/25872)官方支持osc52了<br>
> 
>UPDATE 2024.5.28:<br>
> 插件还没用上一个星期就用上10.0了.......
# 先决条件
1、终端支持osc52<br>
2、使用neovim<br>
笔者配置:
<img src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnVGNOdGk4QnpVWnFXSU1rP2U9d29SeXBh.png" alt="" >
# 实践
## 1-配置终端
笔者使用的终端是suckless的`st`，但是并非默认开启osc52功能，需要修改源码重新编译以开启[功能](https://git.suckless.org/st/commit/a2a704492b9f4d2408d180f7aeeacf4c789a1d67.html)。对于其他终端，vim-oscyank的作者给了一份表以[参考](https://github.com/ojroques/nvim-osc52)

|Terminal 	|OSC52 support|
|:---|:---|
|alacritty| 	yes|
|contour| 	yes|
|far2l| 	yes
|foot| 	yes|
|gnome terminal (and other VTE-based terminals)| 	not yet|
|hterm| 	yes|
|iterm2| 	yes|
|kitty| 	yes|
|konsole| 	not yet|
|qterminal| 	not yet|
|rxvt| 	yes|
|st| 	yes (but needs to be enabled, see here)|
|terminal.app| 	no, but see workaround|
|tmux| 	yes|
|urxvt| 	yes (with a script, see here)|
|wezterm| 	yes|
|windows terminal| 	yes|
|xterm.js (Hyper terminal)| 	not yet|
|zellij| 	yes|

> hexo的表格...晚点研究一下

照着各家终端配置完测试功能:
```bash
echo -ne "\033]52;c;$(echo -n hello | base64)\a"
```
复制以下命令到终端并执行，检查粘贴结果是否为hello，否则步骤未完成

## 2-配置lazy.nvim
`lazy.nvim`是neovim的插件管理器，并非那个vim发行版，注意分别

安装很简单，只需要在nvim的配置文件`~/.config/nvim/init.lua`或`~/.config/nvim/init.vim`加上那么一段代码就能完成安装并配置(前提是有可用的网络):
```lua
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not (vim.uv or vim.loop).fs_stat(lazypath) then
  vim.fn.system({
    "git",
    "clone",
    "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable", -- latest stable release
    lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)
require("lazy").setup({})
```

当然也能提前把这个仓库克隆到对应路径或者使用包管理例如AUR相关的工具安装到相应路径，这个路径得翻翻XDG定义和neovim的文档

重新打开nvim后等待一会，弹出正常界面`:checkhealth lazy`即可验证插件工作情况

## 3-安装插件
安装插件这一步可以非常简单，前提是科学上网，比如说lazy.nvim仓库md的例子。编辑上一节配置文件的最后一行:
```lua
require("lazy").setup({
  "folke/which-key.nvim",		      -- 1、github仓库的名字，下次打开nvim会拉取仓库安装
  { "folke/neoconf.nvim", cmd = "Neoconf" },  -- 2、第二各插件，同上
  "folke/neodev.nvim",                        -- 3、第三个插件，同上
})
-- lazy.nvim仓库内含详细说明，请务必花半个小时了解lua
```

自动安装插件过程:
<img src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnVFp1Z05PMk9TQVAtai1JP2U9VUZlU2R0.png" alt="" >

手动安装当然也可以，只要nvim能找到对应位置，并且按照插件手册写好在配置文件中写好该插件的配置；下面演示手动安装

以[nvim-osc52](https://github.com/ojroques/nvim-osc52)为例，这是仓库的目录结构:
```bash
➜  nvim-osc52 git:(main) tree .
.
├── LICENSE
├── README.md
└── lua
    ├── osc52
    │   └── base64.lua
    └── osc52.lua

3 directories, 4 files


```
很明显，这里面的文件能全部直接放进`~/.config/nvim/`这个目录(当然许可证、md、git文件可以忽略)，这样nvim就知道这里有插件了
> 这种做法自己瞎折腾的，直接把插件当作模块，详见`:h lua-guide-modules`
>
> 使用`:h packadd`参考官方手册受动安装插件正解

但是这样似乎就把lazy.nvim晾在一边了，所以可以在刚刚写好的nvim配置文件中添加一行:
```lua
require("lazy").setup({
{"nvim-lualine/lualine.nvim",dependencies = { 'nvim-tree/nvim-web-devicons' }},
 -- 添加下面这一行以指明插件所在目录
{name="osc52",dir="~/.config/nvim/plugins/nvim-osc52"},
	})

```
然后把整个插件仓库放在刚刚指明的插件目录的位置中即可识别:
```bash
➜  nvim tree .
.
├── init.lua
├── lua
└── plugins
    └── nvim-osc52
        ├── LICENSE
        ├── README.md
        └── lua
            ├── osc52
            │   └── base64.lua
            └── osc52.lua
```
之后执行`:Lazy`检查该路径是否被加载
> 记得依插件说明修改或添加配置文件
### 3.1-部署nvim-osc52
假设上一步只是放置了这个仓库到nvim配置文件中lazy.nvim指明的插件目录而并没有对其配置，那插件自然是没办法常运行的，这时候就要把插件的配置写进nvim的配置文件中

- 首先在init.lua添加这一段:
```lua
vim.keymap.set('n', '<leader>c', require('osc52').copy_operator, {expr = true})
vim.keymap.set('n', '<leader>cc', '<leader>c_', {remap = true})
vim.keymap.set('v', '<leader>c', require('osc52').copy_visual)
```
添加完毕即可使用nvim-osc52这个插件。使用方法为命令模式中按顺序按下`\cc`复制当前行，视图模式中按顺序按下`\c`复制选定文本

- 还能指定将复制内容放到nvim的`+`寄存器:
```lua
function copy()
  if vim.v.event.operator == 'y' and vim.v.event.regname == '+' then
    require('osc52').copy_register('+')
  end
end

vim.api.nvim_create_autocmd('TextYankPost', {callback = copy})
```

- 当然也能将osc52作为nvim的剪切板，但是要先注释掉上面**`+`寄存器**的相关内容:
```lua
local function copy(lines, _)
  require('osc52').copy(table.concat(lines, '\n'))
end

local function paste()
  return {vim.fn.split(vim.fn.getreg(''), '\n'), vim.fn.getregtype('')}
end

vim.g.clipboard = {
  name = 'osc52',
  copy = {['+'] = copy, ['*'] = copy},
  paste = {['+'] = paste, ['*'] = paste},
}

-- Now the '+' register will copy to system clipboard using OSC52
vim.keymap.set('n', '<leader>c', '"+y')
vim.keymap.set('n', '<leader>cc', '"+yy')
```
这时候就能愉快地使用vim命令复制到osc52的剪切板了，缺点是nvim外部复制的内容需要在插入模式下`shift+control+c`才能粘贴进nvim。用vim命令能够复制线面会弹出osc52的提示
 
## 4-重新规划配置文件(可选)
所有插件的配置全扔一起感觉有点乱。笔者眼有点花所以需要做点特别的配置

首先是目录结构:
```bash
 ➜  nvim tree .
.
├── init.lua
├── init.vim.bak
├── lazy-lock.json
├── lua
│   ├── individual
│   │   ├── lualine_conf.lua
│   │   └── nvim-osc52_conf.lua
│   ├── lazy_conf.lua
│   └── opt_conf.lua
└── plugins
    └── nvim-osc52
        ├── LICENSE
        ├── README.md
        └── lua
            ├── osc52
            │   └── base64.lua
            └── osc52.lua

7 directories, 11 files
 
 ```
 `init.lua`仅仅声明后续用到的文件:
 ```lua
require('lazy_conf')
require('opt_conf')
 ```
 `lazy_conf.lua`包含lazy.nvim本身的配置和声明各个插件的配置:
 ```lua
 local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not (vim.uv or vim.loop).fs_stat(lazypath) then
  vim.fn.system({
    "git",
    "clone",
    "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable", -- latest stable release
    lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)
require("lazy").setup({
{"nvim-lualine/lualine.nvim",dependencies = { 'nvim-tree/nvim-web-devicons' }},
{name="osc52",dir="/home/baka/.config/nvim/plugins/nvim-osc52"},
	})

-- IMPORT CONFIG FOR EACH PLUGINS
require('individual.nvim-osc52_conf') -- `nvim/lua/individual/nvim-osc52.lua`包含osc52配置
require('individual.lualine_conf')    -- `nvim/lua/individual/lualine_conf.lua`包含lualine配置
 ```
 `opt.lua`包含nvim的option:
 ```lua
vim.opt.number = true
vim.opt.relativenumber = true
vim.opt.cursorline = true
vim.highlight.priorities.syntax = 50
vim.opt.clipboard:append {'unnamedplus'}
vim.opt.hidden = true
vim.opt.tabstop = 2
vim.opt.shiftwidth = 2
vim.opt.softtabstop = 2
vim.opt.smartindent = true
 ```
对于其他插件各自的配置文件统一放在`nvim/lua/individual/`文件夹。虽然配置完得去`lazy_conf.lua`声明一下，但是不会眼花(感觉可以用lua写脚本实现自动声明)
