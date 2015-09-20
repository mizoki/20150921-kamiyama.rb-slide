---
title: "普段使っているMacの調子が悪くなったので、WindowsでRubyの開発環境を整える"
author: mizoki
date: September 21, 2015
---

# Macの調子が悪くなったので、WindowsでRubyの開発環境を整える

普段はMacを使って、開発をしていますが、Macの調子が悪くなってきたので、予備のWindowsに環境を構築しました。

今回使用するソフトウェアは以下のとおりです。

<ul>
  <li>Ansible</li>
  <li>git</li>
  <li>Virtualbox</li>
  <li>Vagrant</li>
</ul>

# ソフトウェアのインストール

以下のソフトウェアを公式サイトからダウンロードしてインストールします。

<ul>
  <li>Git</li>
  <ul>
  <li>[http://git-scm.com/](http://git-scm.com/)</li>
  </ul>
  <li>Virtualboxのインストール</li>
  <ul>
  <li>[https://www.virtualbox.org/wiki/Downloads](https://www.virtualbox.org/wiki/Downloads)</li>
  </ul>
  <li>Vagrant</li>
  <ul>
  <li>[https://www.vagrantup.com/](https://www.vagrantup.com/)</li>
  </ul>
</ul>

# 各ソフトウェアの役割

## Ansible

環境の構築に利用する

## Git

Ansibleの設定ファイルのバージョン管理と`Git Bash`からsshを使用するために利用する

## Virtualbox, Vagrant

仮想マシンの構築に利用する

```shell
$ vagrant up      # 仮想マシンの起動

$ vagrant ssh     # 仮想マシンへsshで接続

$ vagrant halt    # 仮想マシンのシャットダウン

$ vagrant destroy # 仮想マシンの削除
```

# Ansibleの構成

使用するAnsibleのplaybookの構成は以下のとおりです。

<ul>
  <li>[https://github.com/mizoki/ansible-playbook](https://github.com/mizoki/ansible-playbook)</li>
</ul>

```
ansible-playbook
├── README.md
├── vagrant-centos
│   ├── hosts
│   ├── playbook.yml
│   └── roles
│       ├── common
│       │   └── tasks
│       │       └── main.yml
│       ├── ghq
│       │   └── tasks
│       │       └── main.yml
│       ├── git
│       │   └── tasks
│       │       └── main.yml
│       ├── haskell
│       │   └── tasks
│       │       └── main.yml
│       ├── peco
│       │   └── tasks
│       │       └── main.yml
│       ├── ruby
│       │   └── tasks
│       │       └── main.yml
│       ├── slidy-builder
│       │   └── tasks
│       │       └── main.yml
│       ├── tig
│       │   └── tasks
│       │       └── main.yml
│       ├── tmux
│       │   └── tasks
│       │       └── main.yml
│       ├── vim
│       │   └── tasks
│       │       └── main.yml
│       └── zsh
│           └── tasks
│               └── main.yml
└── virtualbox
    └── centos
            └── Vagrantfile
```

# Vagrantfile

仮想マシンの設定は以下のとおりです。

今回はCentOS 7を利用します。

本来、Ansibleはローカルからsshで接続できれば、設定される側には何もインストールする必要がありませんが、pythonを入れたりするのが面倒なので、CentOSにAnsibleをインストールして、自分で自分の設定をしてもらいます。

そのために、Windows側のAnsibleの設定ファイルがCentOSからも見えるように、共有フォルダの設定を行っています。

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "puppetlabs/centos-7.0-64-nocm"

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  config.vm.network "private_network", ip: "192.168.33.10"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"
  config.vm.synced_folder "../../vagrant-centos", "/ansible", mount_options: ["dmode=755","fmode=644"]

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  config.vm.provision "shell", inline: <<-SHELL
    if ! [ `which ansible` ]; then
      yum update -y
      yum install -y epel-release
      yum install -y ansible
    fi
    ansible-playbook -i /ansible/hosts /ansible/playbook.yml
  SHELL
end
```

# Ansibleの設定

## playbook.yml

```yaml
- hosts: 127.0.0.1
  connection: local
  sudo: yes
  roles:
    - common
    - ghq
    - git
    - haskell
    - peco
    - ruby
    - slidy-builder
    - tig
    - tmux
    - vim
    - zsh
```

## hosts

```
127.0.0.1 ansible_connection=local
```

# roles/common/tasks/main.yml

rolesの中身をいくつか紹介します。

```yaml
- name: system update
  yum: name=* state=latest      # nameにアスタリスクを指定するとyum updateを実行する

- name: install some package for development
  yum: name={{item}} state=latest
  with_items:
    - autoconf
    - automake
    - curl-devel
    - expat-devel
    - gettext
    - gettext-devel
    - gpm-libs
    - libacl-devel
    - libevent
    - libevent-devel
    - lua
    - lua-devel
    - ncurses
    - ncurses-devel
    - ncurses-libs
    - openssl-devel
    - pcre
    - pcre-devel
    - perl-ExtUtils-Embed
    - perl-core
    - perl-libs
    - python-devel
    - ruby
    - ruby-devel
    - unzip
    - zlib-devel
    - git
    - colordiff
    - nkf

- name: clone repository of luajit
  become: yes
  become_user: vagrant
  git:
    repo: http://luajit.org/git/luajit-2.0.git
    dest: /home/vagrant/local/src/luajit-2.0

- name: build luajit
  shell: make && make install
  args:
    chdir: /home/vagrant/local/src/luajit-2.0

- name: clone repository of my dotfiles
  become: yes
  become_user: vagrant
  git:
    repo: https://github.com/mizoki/dotfiles.git
    dest: /home/vagrant/repos/github.com/mizoki/dotfiles

- name: check zshrc
  shell: test -f /home/vagrant/.zshrc
  register: result
  ignore_errors: true
  changed_when: false

- name: add symbolic link of my dotfiles
  become: yes
  become_user: vagrant
  shell: /home/vagrant/repos/github.com/mizoki/dotfiles/setup.sh
  when: result|failed
  args:
    chdir: /home/vagrant/repos/github.com/mizoki/dotfiles

- name: check setting of $PATH
  shell: cat /home/vagrant/.zshenv | grep -q 'export PATH=/home/vagrant/local/bin:$PATH'
  register: result
  ignore_errors: true
  changed_when: false

- name: set environment variable ($PATH)
  become: yes
  become_user: vagrant
  shell: echo 'export PATH=/home/vagrant/local/bin:$PATH' >> /home/vagrant/.zshenv
  when: result|failed

- name: check setting of timezone
  shell: test -f /etc/localtime
  register: result
  ignore_errors: true
  changed_when: false

- name: set timezone
  shell: ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
  when: result|failed
```

# roles/vim/tasks/main.yml

次にこれがなければ何もできないという超重要なソフトウェアであるvimの設定です。

```yaml
- name: clone repository of vim
  become: yes
  become_user: vagrant
  git:
    repo: https://github.com/vim/vim.git
    dest: /home/vagrant/repos/github.com/vim/vim

- name: build vim
  become: yes
  become_user: vagrant
  shell: |
    ./configure \
      --with-features=huge \
      --with-compiledby="mizoki <h.mizoki@gmail.com>" \
      --prefix=/home/vagrant/local \
      --with-x \
      --with-gnome \
      --enable-xim \
      --enable-fail-if-missing \
      --enable-luainterp=yes \
      --enable-perlinterp=yes \
      --enable-pythoninterp=yes \
      --enable-rubyinterp=yes \
      --with-lua-prefix=/usr \
      --with-luajit \
      --enable-multibyte
    make
    make install
    make clean
  args:
    chdir: /home/vagrant/repos/github.com/vim/vim

- name: add loading setting of luajit
  shell: |
    echo "/usr/local/lib/" > /etc/ld.so.conf.d/luajit.conf
    ldconfig

- name: clone repository of neobundle.vim
  become: yes
  become_user: vagrant
  git:
    repo: https://github.com/Shougo/neobundle.vim.git
    dest: /home/vagrant/.vim/bundle/neobundle.vim
```

# はまったところ

## Windowsでは、共有フォルダへのアクセス権が777になる

`hosts`ファイルに実行権限があると、Ansibleの実行時にエラーになる。

```shell
    ansible-playbook -i /ansible/hosts /ansible/playbook.yml
```

### 解決策

共有フォルダの指定時に`mount_options`を指定して、アクセス権を設定する

#### Mac（Linux）ならこれでOK

```ruby
  config.vm.synced_folder "../../vagrant-centos", "/ansible"
```

#### Windowsの場合

```ruby
  config.vm.synced_folder "../../vagrant-centos", "/ansible", mount_options: ["dmode=755","fmode=644"]
```

---

## vagrant upでmountエラーが発生する

2回目以降の、`vagrant up`の実行時に以下のエラーが発生する

```
Failed to mount folders in Linux guest. This is usually because
the "vboxsf" file system is not available. Please verify that
the guest additions are properly installed in the guest and
can work properly. The command attempted was:

mount -t vboxsf -o uid=`id -u vagrant`,gid=`getent group vagrant | cut -d: -f3`,dmode=755,fmode=644 ansible /ansible
mount -t vboxsf -o uid=`id -u vagrant`,gid=`id -g vagrant`,dmode=755,fmode=644 ansible /ansible

The error output from the last command was:

/sbin/mount.vboxsf: mounting failed with the error: No such device
```

### 解決策

<ul>
  <li>vagrantでmountエラーの解決方法 - Qiita</li>
  <ul>
    <li>[http://qiita.com/osamu1203/items/10e19c74c912d303ca0b](http://qiita.com/osamu1203/items/10e19c74c912d303ca0b)</li>
  </ul>
</ul>

#### ホストマシンのシェル

```shell
$ vagrant ssh
```

#### 仮想マシンのシェル

vboxのリビルドを行う

```shell
$ sudo /etc/init.d/vboxadd setup
```

#### 仮想マシンの再起動

```shell
$ vagrant halt
$ vagrant up
```

もしくは

```shell
$ vagrant reload
```

# TODO

- リポジトリのクローン時に変更がなかった場合は、ビルドしたくない
- ユーザー名やgitリポジトリのクローン先を変数にして、VPSなどVagrant以外のセットアップにも使えるようにしたい
- Serverspecで構成をテストしたい
- 特に時間短縮のためにも、ビルドしなくていいときにはビルドしないようにしたいけれど、誰かいい方法を知っていたら教えてください。

# まとめ

今回は、今までシェルスクリプトでやっていたことを、単純に置き換えただけのものです。

Ansibleはもっと色々なことができるみたいなので、もっと設定を詰めていきたいです。

ですが、これだけでもWindowsでも簡単に開発が始められるようになりました。

---

ただ、今日持ってきたPCは、Arch Linuxとのデュアルブートで普段はArch Linuxを使ってます。

---

ご清聴ありがとうございました。
