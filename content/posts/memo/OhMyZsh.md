# Zsh

安装 zsh:

```sh
sudo apt install zsh
```

设置 zsh 为 default shell:

```sh
chsh -s $(which zsh)
echo $SHELL # 检查当前shell
```

# Oh My Zsh

安装 Oh My Zsh:

```sh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

## Theme

主题安装到`~/.oh-my-zsh/custom/theme`目录下。

使用网上推荐的 powerlevel10k:

```sh
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

在.zshrc 中配置主题：

```sh
ZSH_THEME="powerlevel10k/powerlevel10k"
```

## Plugins

主题安装到`~/.oh-my-zsh/custom/plugin`目录下。

### zsh-autosuggestions

智能补全命令的插件，按下右键可以快速采用建议。

```sh
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

### zsh-syntax-highlighting

智能检查输入命令语法的插件。

```sh
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

### z

oh-my-zsh 自带的智能跳转文件夹的插件。

用法：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241107223655.png)

可以模糊补全，不用输入整个文件夹的名称。

# Reference

Download https://github.com/YC-Xiang/dotfiles 中的 `.zshrc` 和 `.p10k.zsh`

https://www.haoyep.com/posts/zsh-config-oh-my-zsh/
