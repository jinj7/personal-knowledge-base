# 1. install ruby

```sh
# Install Homebrew
$ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
 
$ brew install ruby
 
$ echo 'export PATH="/usr/local/opt/ruby/bin:$PATH"' >> ~/.zshrc
$ ruby -v
```

&nbsp;


# 2. install jekyll

```sh
$ gem install bundler jekyll
 
$ echo 'export PATH="$HOME/.gem/ruby/X.X.0/bin:$PATH"' >> ~/.zshrc
$ gem env
 
$ gem install bundler:1.17.1
 
$ bundle install
```

如果报错bundler版本不合适：

```sh
$ gem install bundler:1.17.1
```

如果报错缺少gems：

```sh
$ bundle install
```

最终在localhost启动jekyll：

```sh
$ bundle exec jekyll serve
```
