---
title: neovim伪ide改造
tags:
  - nvim
categories:
  - - linux
    - nvim
date: 2024-08-25 03:40:46
---

最近计划用python写一些工具，刚好近几个月以来常用的电脑没装ide和某个有重量级且有无可估量潜力的editor。然而经过平铺窗口管理器和vim的调教后双手已经无法脱离键盘了...故打算重整一下nvim,令其能够在尽可能简单的配置下实现ide的常规功能

效果:
<img src="https://telegraph.7cmb.com/file/3c531b5595f845d8062a4.png" alt="" >

# 1-需求&对应插件

算上以前装的插件，总结下来需求有以下几点:
- [tab键自动补全，并适配python3，并配备语法检查](https://github.com/neoclide/coc.nvim)
- [缩进空白竖线](https://github.com/lukas-reineke/indent-blankline.nvim)
- [有一个边栏给我切换文件(在笔者这里buffer最场景的一集)](https://github.com/nvim-tree/nvim-tree.lua)
- [osc52yank](https://github.com/ojroques/nvim-osc52)(我认为目前插件比起官方方法更稳定)
- [lualine美化底边](https://github.com/nvim-lualine/lualine.nvim)

我使用的neovim包管理为[lazy.nvim](https://github.com/folke/lazy.nvim),所以以下内容只对lazy.nvim负责

> 其实nvim已经内置lsp client了，管理lsp可以all in mason。笔者选择coc的原因纯粹是因为coc是先找到的工具，以后再去考虑mason
> 
> 编译完成的coc体积高达300m,太悲伤了
>
> [关于mason和coco](https://github.com/neoclide/coc.nvim/discussions/4866)

# 2-管理Lazy.nvim

对于Lazy.nvim[官方推荐的做法](https://lazy.folke.io/usage/structuring)，我并不是很感冒，原因有三个:
- 我想在只一个文件里管理我需要开启的插件及插件对应的配置，即使需要人工干预得更多
- 一个一个管理更加直观
- 因为网络原因，我更喜欢将仓库手动clone下来，再手动指定插件目录，以体积换取配置迁移的便利(所以下面有个插件仓库)


所以这是我的配置结构:
```bash
➜  nvim tree -L 2 .
.
├── init.lua ‘# 默认配置文件’
├── init.vim.bak
├── lazy-lock.json
├── lua
│   ├── lazy_conf.lua ‘# Lazy.nvim配置文件’
│   ├── opt_conf.lua  ‘# nvim 配置文件’
│   └── plugins_conf  ‘# 插件配置文件夹’
└── plugins ‘# 插件仓库’
    ├── coc.nvim
    ├── indent-blankline.nvim
    ├── nvim-tree.lua
    ├── nvim.osc52
    └── sidebar.nvim

```

`init.lua`的唯一的作用就是用`require`调用真正的配置文件:
```lua
---@NVIM PLUGINS AND PLUGINS CONF HEAD
---@PATH TO nvim/lua/lazy_conf.lua
require('lazy_conf')

---@NVIM OPTIONS
---@PATH TO nvim/lua/opt_conf.lua
require('opt_conf')

```

`nvim/lua/opt_conf.lua`使用lua重写我的vim配置文件:
```lua
vim.opt.number = true
vim.opt.relativenumber = true
vim.opt.cursorline = true
vim.highlight.priorities.syntax = 50 
vim.opt.clipboard:append {'unnamedplus'}
-- vim.opt.hidden = true
vim.opt.tabstop = 2
vim.opt.shiftwidth = 2
vim.opt.softtabstop = 2
vim.opt.expandtab = true
vim.opt.smartindent = true
vim.cmd.colorscheme('desert')
vim.cmd.colorscheme('habamax')

---@ OFFICAL OSC52
---@ 内置OSC52
--vim.g.clipboard = {
--	name = 'OSC 52',
--	copy = {
--		['+'] = require('vim.ui.clipboard.osc52').copy('+'),
--		['*'] = require('vim.ui.clipboard.osc52').copy('*'),
--	},
--	paste = {
--		['+'] = require('vim.ui.clipboard.osc52').paste('+'),
--		['*'] = require('vim.ui.clipboard.osc52').paste('*'),
--	},
--}
---END
```

`nvim/lua/lazy_conf.lua`用于配置lazy.nvim:
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

---@ MANAGE PLUGINS
---@ 管理插件
require("lazy").setup({
{"nvim-lualine/lualine.nvim",dependencies = { 'nvim-tree/nvim-web-devicons' }},
{name="osc52",dir="/home/baka/.config/nvim/plugins/nvim.osc52"},
{name="coc-install",dir="/home/baka/.config/nvim/plugins/coc.nvim"},
{name="indent-blankline",dir="/home/baka/.config/nvim/plugins/indent-blankline.nvim",main="ibl", --[[@module "ibl" @type ibl.config--]] opts={},},
{name="nvim-tree",dir="/home/baka/.config/nvim/plugins/nvim-tree.lua",dependencies={"nvim-tree/nvim-web-devicons",},opt={}},
-- {name="sidebar",dir="/home/baka/.config/nvim/plugins/sidebar.nvim"},
	})

---@ IMPORT PLUGINS CONFIG
---@ 导入插件配置文件
require('plugins_conf.nvim-osc52_conf')
require('plugins_conf.lualine_conf')
require('plugins_conf.coc-nvim_conf')
require('plugins_conf.nvim-tree_conf')
require('plugins_conf.nvim-tree-keysmapping_conf')
-- require('plugins_conf.sidebar_conf')
-- require('plugins_conf.lualine-evil_conf')
```

## nvim-tree文档可能的一点typo

根据该项目的[安装文档](https://github.com/nvim-tree/nvim-tree.lua/wiki/Installation)没法成功安装:
```lua
return {
  "nvim-tree/nvim-tree.lua",
  version = "*",
  lazy = false,
  dependencies = {
    "nvim-tree/nvim-web-devicons",
  },
  config = function()
    require("nvim-tree").setup {} ---@ 疑似这里少了括号
  end,
}
```

参考[lazy.nvim的这个章节](https://lazy.folke.io/developers)，应该能得出两种改法:
```lua
---@ 改法1:
{
  "nvim-tree/nvim-tree.lua",
  version = "*",
  lazy = false,
  dependencies = {
    "nvim-tree/nvim-web-devicons",
  },
  config = function()
    require("nvim-tree").setup ({}) ---@ 把括号加上
  end,
}
--- END

---@ 改法2:
{"nvim-tree/nvim-tree.lua",dependencies={"nvim-tree/nvim-web-devicons",},opt={}}
---

```

# 3-绑定快捷键

> 本节参考:
>
> https://neovim.io/doc/user/api.html#nvim_exec2()
>
> https://neovim.io/doc/user/lua.html#vim.keymap.set()

上述nvim-tree插件的边栏呼出的话需要执行`:NvimTreeToggle`。尝试了该仓库doc的快捷键绑定方法，发现只能边栏在已经呼出的情况下使用。但是我需用`control+l`随时呼出关闭，那么在该插件的配置文件中添加下面行:
```lua
---@ GLOBAL SELF MAPPING 
---@ 两种方法
-- vim.keymap.set('n', '<C-l>',function() vim.api.nvim_exec2("NvimTreeToggle",{output}) end)
vim.keymap.set('n', '<C-l>',function() vim.api.nvim_cmd({cmd="NvimTreeToggle"},{output}) end)
```
