---
title: 在一个新的linux系统下搭建hexo开发环境
date: 2021-09-25 20:55:53
tags:
---



# 以mint系统为例

# 安装typora

```shell
# or run:
# sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys BA300B7755AFCFAE
wget -qO - https://typora.io/linux/public-key.asc | sudo apt-key add -
# add Typora's repository
sudo add-apt-repository 'deb https://typora.io/linux ./'
sudo apt-get update
# install typora
sudo apt-get install typora
```

# 安装git、nodejs、npm

```shell
#sudo apt install nodejs，默认源的node版本教老，不采用
sudo apt install npm
# Using Ubuntu，参考文档https://nodejs.org/en/download/package-manager/#debian-and-ubuntu-based-linux-distributions
curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
sudo apt-get install -y nodejs

```

# 安装hexo

```shell
#实现非root用户安装不报错，需要进行如下配置
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
#In your preferred text editor, open or create a ~/.profile file and add this line:
export PATH=~/.npm-global/bin:$PATH
source ~/.profile
#安装hexo
npm install hexo-cli -g
```

# git拉取源代码

```shell
#配置git拉去和推送权限
#生成ssh密钥对，参看文档:https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#
ssh-keygen -t rsa -b 4096 -C "sandszy@126.com"
#将生成的公钥注册到github的个人设置的“SSH and GPG keys”菜单下
cat /home/sand/.ssh/id_rsa.pub
#设置git用户名和邮箱，邮箱会与github邮箱匹配来判断权限
git config --global user.email "sandszy"
git config --global user.email "sandszy@126.com"
#测试权限配置正常
ssh -T git@github.com
#创建代码目录并拉取代码
mkdir -p ~/Github/sandszy/
cd ~/Github/sandszy/
git clone git@github.com:sandszy/sandszy.github.io.git
#检出hexo的源码分支save
git checkout save
```

# 进行hexo的使用

```shell
#第一次checkout源码后需要重新安装hexo相关以来
cd ~/GitHub/sandszy/sandszy.github.io/myBlog
npm install
#安装部署插件
npm install hexo-deployer-git --save
#然后即可进行正常的hexo文档编写工作
hexo new "在一个新的linux系统下搭建hexo开发环境"
hexo g
hexo s
#测试ok后推送

hexo g -d

```





