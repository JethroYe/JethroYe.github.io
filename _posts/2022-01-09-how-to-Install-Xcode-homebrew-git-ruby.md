---
layout: post
title: MacOS 配置ruby环境指引
updated: 2022-01-02 18:00
tag: 
- 翻译
- ruby
- homebrew
- Mac
- Xcode
---

> <big>*世界上80%的有效知识都是英文书写的*</big>  
> <big>由于谷歌机翻结果往往差强人意，所以我准备不定期翻译一些有价值的英文资料或是技术文档。</big>    
> <big>此过程不仅帮助我掌握知识，同时也能为网络社区贡献更多中文力量</big>  
> <big>原文：**[How to install Xcode, Homebrew, Git, a Ruby manager (chruby, rbenv, RVM), Rails, Jekyll on macOS (including M1 Apple Silicon)](https://www.moncefbelyamani.com/how-to-install-xcode-homebrew-git-rvm-ruby-on-mac/?utm_source=stackoverflow)**<big>   

# 正文开始  
**本文更新于2022年1月13日**  

## 帮你在Mac上安装任意的Ruby gem
你是否曾经遇到过可怕的gem权限错误？
```
ERROR: While executing gem ... (Gem::FilePermissionError)
You don't have write permissions for the /Library/Ruby/Gems/2.6.0 directory
```
你是否曾经使用`sudo`安装gem然后导致报错？
```
ERROR: Failed to build gem native extension.
```
你是否一直在努力安装 rails、jekyll、bundler、cocoapods、fastlane 或其他一些 Ruby gem？ 你是否一直在输入一大串命令，并试图一遍又一遍的修复错误？你是否被这些问题困扰了好多天？  
**今天你将终结所有的Ruby问题**  
我在过去的10年一直致力于帮助人们解决这类问题，看看[他们是怎么说的吧](https://www.moncefbelyamani.com/what-they-say/)

## 你的选择
- 使用我的脚本可以帮助你一键配置好全局环境。脚本也可以帮助你的保持你的开发环境安全并且时刻保持最新。一条简单的命令，你就不用再阅读这份指引了
- 花几个小时阅读这份指引，然后手动配置所有环境。并且未来你也需要手动维护你的研发环境

## 使用我的脚本
本脚本目前是免费的，当然我也在持续维护它。如果想要获得免费脚本和高级脚本55%的折扣。请点击加入下面的列表。此折扣仅仅适用与订阅我的读者，在这里你可以读到[读者们对我的脚本和指南的评价](https://www.moncefbelyamani.com/what-they-say)  
译者：请在原文中加入或者订阅

## 终端（以下称为Terminal）
本指南中的许多工作都需要在一个被称为 Terminal 的App 中完成。在Mac 中，通过[Spolight](https://support.apple.com/en-us/HT204014) 可以最快捷的打开Terminal。默认的键盘快捷键commend+space 可以快速打开Spolight。  

## 准备工作 
本文支持的 MacOS 版本:
- Monterey
- Big Sur
- Catalina
老版本的MacOS 上本文可能也适用，但是不保证。如果你遇到任何问题，可以阅读本指南尝试自己解决。如果解决不了，可以订阅[我的付费课程](https://savvycal.com/monfresh/dev-setup?d=15&from=2022-01-09)，我会尽量提供帮助

### 确保你的电脑已经插入电源并且有一个稳定的网络环境
脚本可能需要 15 分钟才能安装所有内容，因此如果你打算在它运行时执行其他操作，请暂时阻止它休眠，直到脚本完成。你可以在系统偏好设置下的电池 -> 电源适配器设置中执行此操作。打开“显示器关闭时防止计算机自动休眠”选项。

### 确保你的MacOS 软件是最新的
单击屏幕左上角的 Apple 图标，然后单击系统偏好设置，然后单击软件更新。如果有任何可用更新，请安装它们。如果你已经拥有 Big Sur 的最新更新，则无需升级到 Monterey。

### 确保你的Homebrew 已经准备好了
如果你的Homebrew 已经安装好了，可以跳过这一节。如果你不确定，可以通过下面的命令检查一下 **/usr/local**目录中是否有内容。如果什么都没有，表示你的电脑上没有Homebrew，你需要安装它
```
ls /usr/local
```
如果是Apple Silicon chip (以下简称M1) 版本的Mac，你需要检查 **/opt/homebrew** 目录  
如果上面的这些目录都不是空的，则表示你的电脑上已经安装了Homebrew。你可以运行下面的命令来检查下你的brew是否正常。  
```
brew doctor
```
正常情况下，你会看到**Your system is ready to brew.** 的提示。如果brew没有准备好，你可能会看到一个或者多个提示。那么接下来的首要工作，就是修好brew 上面的提示。命令行输出的信息可以给你更多的帮助。
```
Warning: A newer Command Line Tools release is available.
Update them from Software Update in System Preferences or run:
  softwareupdate --all --install --force

If that doesn't show you any updates, run:
  sudo rm -rf /Library/Developer/CommandLineTools
  sudo xcode-select --install

Alternatively, manually download them from:
  https://developer.apple.com/download/more/.

```
以下是多种形式的报错提示信息。
```
Warning: Your Command Line Tools are too outdated.
```
```
Warning: Your Command Line Tools (CLT) does not support macOS 11.
It is either outdated or was modified.
Please update your Command Line Tools (CLT) or delete it if no updates are available.
```
如果缺少一些工具，会看到这样的提示
```
Warning: No developer tools installed.
Install the Command Line Tools:
  xcode-select --install
```
Hombrew 也会提供一些更详细的修复指南。仔细阅读指南并且跟着操作。当CLT安装好之后，退出并且重启Terminal。  
如果你遇到一些其他的问题，可以阅读下面的[Homebrew 问题解决指导](TODO:这里的跳转看看怎么实现)
**如果你仍然遇到一些自己无法解决的问题，我的高级脚本也许有用**  
译者：高级脚本可以在原文中获取。

#### 确保电脑中没有RVM，rbenv或者asdf
在所有的ruby 管理器中。我推荐**chruby**，因为它更加稳定也更加方便维护。ruby管理器之间并不兼容，所以你如果你的设备里有RVM，rbenv，或者是asdf，你需要将他们卸载掉。 
#### 检查是否已经有了RVM 和 rbenv
在Terminal中运行这几条命令
```
# 检查是否有rvm
rvm help
# 检查是否有rbenv
rbenv help
# 检查是否有asdf
asdf help
```
如果两条命令输出了任何内容，都表示你已经安装了他们。相反如果返回了 `command not found`，表示没有安装。
卸载rvm 可以通过下面的命令
```
rvm implode
```
然后，需要在下面的文件中删除所有跟rvm 相关的内容。
- `~/.bash_profile`
- `~/.zshrc`
- `~/.zprofile`  

## 安装说明
### 如果你的设备是M1 Mac，你需要阅读下面这段，否则可以直接跳过
如果你使用的是M1 版本的Mac，需要确保Terminal 没有运行在 Rosetta mode （罗赛塔模式）。
运行下面的命令来检查
```
uname -m
```
正常情况下预期会打印出`arm64`，如果它打印了 `x86_4`，那意味着Terminal 运行在罗萨塔模式下。按照下面的操作，可以关闭罗塞塔：
1. 完全退出Terminal
2. 打开Finder
3. 按 shift-command-U 进入 Utilities 文件夹（或从菜单栏中选择“Go”，然后选择 Utilities）
4. 选中Terminal，但是不要启动它。
5. 按Commend+i, 取消选择框中的"Open using Rosetta"
6. 启动Terminal，运行`uname -m`检查其输出是变成了`arm64`

### 关于shell
本指南假设你是用的shell 是 zsh。如果你不确定，可以阅读我的这篇引导[查看你是用的是什么shell](https://www.moncefbelyamani.com/which-shell-am-i-using-how-can-i-switch/). 如果你是用的是bash，那么需要在所有关于`.zshrc` 的步骤中，将其替换为`.bash_profile`

## 步骤1: 安装Homebrew 和 命令行工具(Command Line Tools)
[Homebrew](https://brew.sh/) 是MacOS 上的工具包管理器，可以帮助你方便的安装数百个开源工具。Homebrew 文档中提供了完整的安装说明，但你只需要运行 Homebrew 站点顶部列出的命令就能安装Homebrew
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh"
```
将命令粘贴到Terminal 中，然后回车查看Termial 中输出的引导。例如Homebrew 会提示你输入密码。Homebrew 还会自动安装 Apple 命令行工具，通常它会在后台安装。不过也可能出现变化。请注意窗口输出的任何提示。安装成功后，重启Termianl，运行`brew doctor`检查下你的Homebrew是否安装好了  
如果你看到了`Your system is ready to brew`，可以进入 **步骤2**。否则，需要仔细阅读Homebrew的提示。在M1的Mac 上，Homebrew会要求你在安装后再运行几条命令，例如：
```
echo "eval $(/opt/homebrew/bin/brew shellenv)" >> ~/.zprofile
eval $(/opt/homebrew/bin/brew shellenv)
```
运行完成后，再次运行`brew doctor` 检查Homebrew是否安装正常

## 步骤2: 安装chruby，并且使用ruby-install 安装最新的ruby
安装`chruby` 和 `ruby-install`
```
brew install chruby ruby-install
```
安装ruby 3.1.0， 或者在这个网站上获取最新的版本 [ruby site](https://www.ruby-lang.org/en/)
```
ruby-install ruby-3.1.0
```

安装过程可能会持续几分钟，安装完成后，执行下面的命令设置你的shell自动使用`chruby`，注意区分CPU架构
### Intel Mac
```
echo "source /usr/local/share/chruby/chruby.sh" >> ~/.zshrc
echo "source /usr/local/share/chruby/auto.sh" >> ~/.zshrc
echo "chruby ruby-3.1.0" >> ~/.zshrc
```

### Apple Silicon Mac
```
echo "source /opt/homebrew/opt/chruby/share/chruby/chruby.sh" >> ~/.zshrc
echo "source /opt/homebrew/opt/chruby/share/chruby/auto.sh" >> ~/.zshrc
echo "chruby ruby-3.1.0" >> ~/.zshrc
```
重启Terminal，通过下面的命令检查chruby和ruby是否正常.
```
ruby -v
```
预期会看到 `ruby 3.1.0p0` 或者更新版本的输出

## 步骤3: 安装你想要的gem
恭喜！走到这一步你已经拥有了一个正常运行的ruby环境了。现在可以安装 Rails, Jekyll, cocoapods, fastlane, bundler,或者任何在过去的几个月或者几天你想要安装的gem
**但是，千万不要使用 sudo 去安装任何gem！！！** 
现在你已经拥有了一个合适的Ruby开发环境，你可以安全的通过 `gem install XXXX` 的方法安装任何你想要的gem

## 步骤4: 安装Git
[Git](https://git-scm.com/) 是广大Web开发者都会选择的一个[版本管理工具](https://en.wikipedia.org/wiki/Revision_control)，在Homebrew的加持下，安装 Git 非常简单
```
brew update
brew install git
```
如果你刚刚安装好Homebrew，可以跳过`brew update` 步骤。不过在运行brew安装前执行`brew update` 是一个好习惯，因为Homebrew会有规律的更新。  
而后重启Termianl，通过 `git --version` 来查看git的版本

## 下一步
由于安装过的工具包会规律性的更新，可能会修复一些bug或者更新一些feature，所以，保持开发环境更新也是非常重要的。
当然你们也可以使用我的脚本，简单几个命令和单词，它可以帮助你保持研发环境时刻都是最新的。
脚本通过别名来实现上述的功能。别名就是一个长命令的快捷方式。如果你不熟悉别名，可以阅读我的这篇引导[如何使用别名提高你的工作流效率](https://www.moncefbelyamani.com/create-aliases-in-bash-profile-to-assign-shortcuts-for-common-terminal-commands/)  
自动化也是一个高效工程师的标志。如果你找不到提高速度的方式，那么你重复做的大部分事情都会浪费掉时间。  
当你使用我的脚本时，也可以查看实现代码。从中你可以学习到脚本和自动化相关的知识。

### 译者:
后面的内容是原文作者关于付费脚本和Terminal快捷方式的推广，与本文主旨关系不大，对于配置开发环境也没有什么帮助。就不翻译了。如有需要可以自行阅读[原文](https://www.moncefbelyamani.com/how-to-install-xcode-homebrew-git-rvm-ruby-on-mac/?utm_source=stackoverflow)

