# 1. install on-my-zsh

```sh
$ sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
$ sh -c "$(wget https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)"
$ git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
```

修改~/.zshrc：

```text
ZSH_THEME="agnoster"
```

&nbsp;


# 2. oh-my-zsh插件安装

```sh
$ git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
```

配置~/.zshrc：

```text
plugins=(git zsh-autosuggestions)
```