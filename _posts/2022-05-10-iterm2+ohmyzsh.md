---
layout: post
title: iTerm2+oh-my-zsh打造炫酷终端
date: 2022-05-10
tags: Mac
---


## 安装软件包

- 安装 [iTerm2](http://iterm2.com/downloads.html)

- 安装 [oh-my-zsh](https://ohmyz.sh/)

- 修改默认 shell 为 zsh: `chsh -s /bin/zsh`

- 修改[默认主题](https://github.com/ohmyzsh/ohmyzsh/wiki/themes)：
  - cloud 
  - steeef
  - agnosterzak

```
vim ~/.zshrc
ZSH_THEME="agnoster"
```

## 定制主题

- 下载第三方主题 [powerlevel9k](https://github.com/bhilburn/powerlevel9k.git) 到 oh-my-zsh 放置第三方主题的目录中

```
git clone https://github.com/bhilburn/powerlevel9k.git ~/.oh-my-zsh/custom/themes/powerlevel9k
echo "source $ZSH/custom/themes/powerlevel9k/powerlevel9k.zsh-theme" >> ~/.zshrc
```

- 配置主题

```
vim ~/.zshrc
ZSH_THEME="powerlevel9k/powerlevel9k"
source ~/.zshrc
```

- 安装 [powerline](https://powerline.readthedocs.io/en/latest/installation.html) 插件

```
pip3 install powerline-status
pip install powerline-status
```

- 安装[PowerFonts](https://github.com/powerline/fonts.git)

```
git clone https://github.com/powerline/fonts.git --depth=1
cd fonts && ./install.sh
cd .. && rm -rf fonts
```

- 设置 iTerm2 的字体

具体的操作是 iTerm2 -> Preferences -> Profiles -> Text，在 Font 区域选中 Change Font，选择字体名字带有 for powerline 的

![](/images/pic/powerline.png)


- [色彩预设](https://iterm2colorschemes.com/)

选择 Preference -> Profiles -> Colors ，导入色彩主题，并勾上就可以
  - `schemes`: 放置色彩主题文件
  - `screenshots`:  色彩主题预设的预览图

![](/images/pic/scheme.png)

```
git clone https://github.com/mbadolato/iTerm2-Color-Schemes ~/Downloads/itemcolor
```

- 命令补全插件[zsh-autosuggestion](https://github.com/zsh-users/zsh-autosuggestions)

```
cd ~/.oh-my-zsh/custom/plugins/
git clone https://github.com/zsh-users/zsh-autosuggestions
vi ~/.zshrc
plugins=(
        zsh-autosuggestions
        git
)
source ~/.zshrc
```

- 隐藏用户名和主机名

```
echo "prompt_context() {}" >> ~/.zshrc
source ~/.zshrc
```

- 安装配色方案

  - 在打开的finder窗口中，双击Solarized Dark.itermcolors和Solarized Light.itermcolors即可安装明暗两种配色
  - 进入iTerm2 -> Preferences -> Profiles -> Colors -> Color Presets中根据个人喜好选择这两种配色中的一种即可以

```
mkdir ~/OpenSource && cd ~/OpenSource
git clone https://github.com/altercation/solarized
cd solarized/iterm2-colors-solarized/
open .
```


