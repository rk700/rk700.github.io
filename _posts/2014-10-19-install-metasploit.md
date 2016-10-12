---
title: "安装metasploit"
author: rk700
layout: post
tags:
  - linux
  - secTool
---
首先是ruby1.9，因为版本低，所以从aur里装的。为了方便起见，把PKGBUILD里面涉及到的后缀1.9全部去掉，方便。再给gemrc文件里加上 
`gem: --no-ri --no-rdoc` 

gemrc文件可以通过
<pre>$ strace gem  2>&1 | grep gemrc</pre>
找到在哪里。

还需要bundle来管理gems
<pre>$ gem install bundler</pre>

然后是psql。安装完psql后初始化data: 
<pre>$ sudo -u postgres initdb --locale en_US.UTF-8 -E UTF8 -D '/var/lib/postgres/data'</pre>

运行psql，然后换成postgresql用户，创建用户 
<pre>$ createuser --interactive</pre>

因为pg_hba.conf里设定验证方式是trust，所以没有密码。

然后创建数据库
<pre>$ createdb --owner msfUser msfDatabase</pre>

把metasploit下载下来之后，我们用`bundle install`来安装需要的gem包。装完之后运行说还缺少robots，但robots确实安装了。检查后发现是robots那个文件夹里的东西几乎都是other没有任何权限的，又是属于root用户root组。于是给other加上read权限。

为了方便，再把全部msf*放在bin里
<pre>$ for f in msf*; do _bin="/usr/bin/$f"; echo "ruby $(realpath $f) \"\$@\"" > $_bin; chmod 755 $_bin; done</pre>

然后把psql的帐号给msf。在$HOME/.msf4下面创建一个database.yml: 
<pre>
production:
   adapter: postgresql
   database: msfDatabase
   username: msfUser
   host: 127.0.0.1
   port: 5432
   pool: 75
   timeout: 5
</pre>
