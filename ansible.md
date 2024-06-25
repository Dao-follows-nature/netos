# ansible

简介:ansible **基于Python开发**，实现了批量系统配置、批量程序部署、批量运行命令等功能。ansible只是提供一种框架。ansible不需要在远程主机上安装client/agents，因为它们是**基于ssh来**和远程主机通讯的。

官方文档:[Ansible 文档 — Ansible 文档](https://docs.ansible.com/ansible/latest/index.html)

## **ansible 架构图**

![img](E:\笔记\images\1464965-20200316134640816-1663805235-1699889937982-1.jpg)

`Ansible`:Ansible核心程序。
`HostInventory`:记录由Ansible管理的主机信息，包括端口、密码、ip等。
`Playbooks`:“剧本”YAML格式文件，多个任务定义在一个文件中，定义主机需要调用哪些模块来完成的功能。
`CoreModules`:**核心模块**，主要操作是通过调用核心模块来完成管理任务。
`CustomModules`:自定义模块，完成核心模块无法完成的功能，支持多种语言。
`ConnectionPlugins`:连接插件，Ansible和Host通信使用

## Ansible 特性

基于Python语言实现 

模块化:调用特定的模块完成特定任务，支持自定义模块，可使用任何编程语言写模块

部署简单，基于python和SSH(默认已安装)，agentless，无需代理不依赖PKI（无需ssl） 

安全，基于OpenSSH 

**幂等性:一个任务执行1遍和执行n遍效果一样，不因重复执行带来意外情况,此特性非绝对** 

支持playbook编排任务，YAML格式，编排任务，支持丰富的数据结构 

较强大的多层解决方案 role

## Ansible安装

yum安装:

```bash
yum install epel-release -y
yum install ansible –y
```

编译安装:

```bash
yum -y install python-jinja2 PyYAML python-paramiko python-babel python-crypto
wget https://releases.ansible.com/ansible/ansible-2.9.9.tar.gz
tar -zxf ansible-2.9.9.tar.gz
cd ansible-2.9.9
python setup.py build
python setup.py install
mkdir /etc/ansible
cp -r examples/* /etc/ansible
```

ubuntu 上安装:

```
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

###  注意事项

执行ansible的主机一般称为管理端, 主控端，中控，master或堡垒机 主控端

Python版本需要2.6或以上 

被控端Python版本小于2.4，需要安装python-simplejson 

被控端如开启SELinux需要安装libselinux-python 

windows 不能做为主控端

## ansible配置文件

安装目录如下(yum安装):
　　配置文件目录:/etc/ansible/
　　执行文件目录:/usr/bin/
　　Lib库依赖目录:/usr/lib/pythonX.X/site-packages/ansible/
　　Help文档目录:/usr/share/doc/ansible-X.X.X/
　　Man文档目录:/usr/share/man/man1/

/etc/ansible/ansible.cfg 主配置文件，配置ansible工作特性,也可以在项目的目录中创建此文件, 当前目录下如果也有ansible.cfg,则此文件优先生效,建议每个项目目录下,创建独有的ansible.cfg文件

/etc/ansible/hosts 主机清单

etc/ansible/roles/ 存放角色的目录

**ansible.cfg的优先级**:

可以在配置文件中进行更改和使用，配置文件将按以下顺序进行搜索:

> - `ANSIBLE_CONFIG`（环境变量，如果已设置）
> - `ansible.cfg`（在当前目录中）
> - `~/.ansible.cfg`（在主目录中）
> - `/etc/ansible/ansible.cfg`

Ansible 将处理上述列表并使用找到的第一个文件，所有其他文件都将被忽略。

```bash
[defaults]
#inventory     = /etc/ansible/hosts #主机列表配置文件
#library = /usr/share/my_modules/ #库文件存放目录
#remote_tmp = $HOME/.ansible/tmp #临时py命令文件存放在远程主机目录
#local_tmp     = $HOME/.ansible/tmp #本机的临时命令执行目录
#forks         = 5   #默认并发数
#sudo_user     = root #默认sudo 用户
#ask_sudo_pass = True #每次执行ansible命令是否询问ssh密码
#ask_pass     = True   
#remote_port   = 22
#host_key_checking = False     #检查对应服务器的host_key，建议取消此行注释,实现第一次连
接自动信任目标主机
#log_path=/var/log/ansible.log #日志文件，建议启用
#module_name = command   #默认模块，可以修改为shell模块
[privilege_escalation] #普通用户提权配置
#become=True
#become_method=sudo
#become_user=root
#become_ask_pass=False
```

ansible 任务执行:　

Ansible 系统由控制主机对被管节点的操作方式可分为两类，即`adhoc`和`playbook`

- ad-hoc模式(点对点模式)
  　　使用单个模块，支持批量执行单条命令。ad-hoc 命令是一种可以快速输入的命令，而且不需要保存起来的命令。**就相当于bash中的一句话shell。**
- playbook模式(剧本模式)
  　　是Ansible主要管理方式，也是Ansible功能强大的关键所在。**playbook通过多个task集合完成一类功能**，如Web服务的安装部署、数据库服务器的批量备份等。可以简单地把playbook理解为通过组合多条ad-hoc操作的配置文件。

## inventory 主机清单文件

ansible的主要功用在于批量主机操作，为了便捷地使用其中的部分主机，可以在inventory file中将其 分组命名

默认的inventory file为 /etc/ansible/hosts

**注意: 生产建议在每个项目目录下创建项目独立的hosts文件**

主机清单文件格式 :INI和YAML

inventory文件遵循INI文件风格，中括号中的字符为组名。可以将同一个主机同时归并到多个不同的组 中 此外，当如若目标主机使用了非默认的SSH端口，还可以在主机名称之后使用冒号加端口号来标明 如果主机名称遵循相似的命名模式，还可以使用列表的方式标识各主机

 **Inventory 参数说明**

```bash
ansible_ssh_host #将要连接的远程主机名.与你想要设定的主机的别名不同的话,可通过此变量设置.
ansible_ssh_port #ssh端口号.如果不是默认的端口号,通过此变量设置.这种可以使用 ip:端口
192.168.1.100:2222
ansible_ssh_user #默认的 ssh 用户名
ansible_ssh_pass #ssh 密码(这种方式并不安全,我们强烈建议使用 --ask-pass 或 SSH 密钥)
ansible_sudo_pass #sudo 密码(这种方式并不安全,我们强烈建议使用 --ask-sudo-pass)
ansible_sudo_exe (new in version 1.8) #sudo 命令路径(适用于1.8及以上版本)
ansible_connection #与主机的连接类型.比如:local, ssh 或者 paramiko. Ansible 1.2 以前
默认使用 paramiko.1.2 以后默认使用 'smart','smart' 方式会根据是否支持 ControlPersist, 
来判断'ssh' 方式是否可行.
ansible_ssh_private_key_file #ssh 使用的私钥文件.适用于有多个密钥,而你不想使用 SSH 代理
的情况.
ansible_shell_type #目标系统的shell类型.默认情况下,命令的执行使用 'sh' 语法,可设置为
'csh' 或 'fish'.
ansible_python_interpreter #目标主机的 python 路径.适用于的情况: 系统中有多个 Python, 
或者命令路径不是"/usr/bin/python",比如 \*BSD, 或者 /usr/bin/python 不是 2.X 版本的
Python.之所以不使用 "/usr/bin/env" 机制,因为这要求远程用户的路径设置正确,且要求 "python" 
可执行程序名不可为 python以外的名字(实际有可能名为python26).与
ansible_python_interpreter 的工作方式相同,可设定如 ruby 或 perl 的路径....
```

### 嵌套组

要创建嵌套组，请执行以下操作:

- 在 INI 格式中，使用后缀`:children`
- 在 YAML 格式中，使用条目`children:`

```
[nginx]
192.168.0.1
[tomcat]
192.168.0.2

[webserver:children]
nginx
tomcat
```

### inventory变量

```
[web]
192.168.0.1  vars=name  #指定单个主机变量 变量名=变量值
[web:vars]
varstwo=name   #设置变量组变量，整个组公用变量
```

注意：当主机变量和组变量重复时，主机变量优先。

## YAML语法简介

- 在单一文件第一行，用连续三个连字号“-”开始（可以省略）。

- 使用#号注释代码
- 缩进必须是统一的，不能空格和tab混用
- 缩进的级别也必须是一致的，同样的缩进代表同样的级别，YAML文件内容是区别大小写的，key/value的值均需大小写敏感
- 多个Key/value可同行写也可换行写，同行使用，分隔。
- value可以是个字符串，也可以是另一个列表
- YAML文件扩展名通常为yaml或yml
- 一个name只能包括一个task



## ansible模块

### 模块帮助查询

```  
ansible-doc [模块名称]  #详细的模块描述手册
ansible-doc -s [模块名称]  #只包含模块参数用法的模块描述手册
```

command、shell、raw和script这四个模块的作用和用法都类似，都用于远程执行命令或脚本：

-  command模块：执行简单的远程shell命令，但不支持解析特殊符号< > | ; &等，比如需要重定向时不能使用command模块，而应该使用shell模块
-  shell模块：和command相同，但是支持解析特殊shell符号
-  raw模块：执行底层shell命令。command和shell模块都是通过目标主机上的python代码启动/bin/sh来执行命令的，但目标主机上可能没有安装python，这时只能使用raw模块在远程主机上直接启动/bin/sh 来执行命令，通常只有在目标主机上安装python时才使用raw模块，其他时候都不使用该模块
-  script模块：在远程主机上执行脚本文件

Ansible 中绝大多数的模块都具有幂等特性，意味着执行一次或多次不会产生副作用。但是 shell、command、raw、script 这四个模块默认不满足幂等性，所以操作会重复执行，但有些操作是不允许重复执行的。例如 mysql 的初始化命令 mysql_install_db，逻辑上它应该只在第一次配置的过程中初始化一次，其他任何时候都不应该再执行。所以，每当使用这 4 个模块的时候都要在心中想一想，重复执行这个命令会不会产生负面影响。



### yum 模块 （安装软件) 不建议用

```
name:必须参数，用于指定需要管理的软件包（要安装软件的名字）
State:用于指定软件包的状态 ，默认值为:present。其中 installed 与present 等效，latest 表示安装 yum 中最新的版本，absent 和 removed 等效，表示删除对应的软件包。
disable_gpg_check:用于禁用对 rpm 包的公钥 gpg 验证。默认值为 no。
enablerepo:用于指定安装软件包时临时启用的 yum 源。
disablerepo:用于指定安装软件包时临时禁用的 yum 源。
```

### package 模块（安装软件）

`package` 模块用于在目标主机上管理软件包，可以安装、升级、删除或检查软件包的状态。该模块支持多个操作系统和软件包管理器，例如 apt、yum、dnf、zypper 等

```
name：指定要安装或管理的软件包的名称。
state：指定软件包的状态，可以是 present（安装）、absent（删除）、latest（升级到最新版本）等。
update_cache：是否更新软件包缓存，默认为 no。当设置为 yes 时，Ansible 将尝试更新软件包缓存。
cache_valid_time：指定软件包缓存的有效时间，默认为不检查更新。
allow_downgrade：是否允许降级软件包，默认为 no。当设置为 yes 时，Ansible 在升级软件包时允许降级。
autoremove：是否自动删除不再需要的软件包，默认为 no。当设置为 yes 时，Ansible 将自动删除不再需要的软件包。
deb：在 Debian 系统上指定要安装的软件包的 URL。
update_cache：是否更新软件包缓存，默认为 no。当设置为 yes 时，Ansible 将尝试更新软件包缓存。
```



### unarchive 模块（解压包）

```
copy:默认为yes，当copy=yes，那么拷贝的文件是从ansible主机复制到远程主机上的，如果设置为copy=no，那么会在远程主机上寻找src源文件。
src:源路径，可以是ansible主机上的路径，也可以是远程主机上的路径或远程URL，如果是远程主机上的路径，则需要设置copy=no（解压的包在哪里）
dest:远程主机上的目标路径(解压到哪里)
mode: 设置解压缩后的文件权限
```

### Script 模块 （在远程主机上运行ansible服务器上的脚本无需执行权限）

无参数直接指定远程主机的sh脚本即可

### archive 模块（压缩模块）

```
Path:要压缩的文件或目录
dest:  压缩后的文件
```

### cron模块（创建任务计划)

```
name:任务计划名称
user:执行任务计划的用户（多用于环境变量）
minute:代表“分钟”
hour:代表“小时”
day:代表“日期”（1-31）
month:代表“月份”
weekday:代表“周“（0-6或1-7）
state:指定任务计划为present为默认差数、当为absent代表删除当前的任务计划
job:任何任务计划所执行的命令。
backup:是否备份之前的任务计划
```

### service（服务模块）

```
enabled :是否开机自动启动，取值为True或者false
name: 服务名称
state: 取值有started(启动)，stopped（停止），restarted（重新启动），reloaded（重新加载）
```

### copy模块（将本地或远程机器上的文件拷贝到远程主机上）

```
Backup:在覆盖之前将原文件备份，备份文件包含时间信息，yes为备份默认不备份。
Src:将本地路径复制到远程服务器; 可以是绝对路径或相对的。如果是一个目录，它将被递归地复制。如果路径以/结尾，则只有该目录下内容被复制到目的地，如果没有使用/来结尾，则包含目录在内的整个内容全部复制。
content:当不使用src指定拷贝的文件时，可以使用content直接指定文件内容，src与content两个参数必有其一，否则会报错。
Dest:目标绝对路径。如果src是一个目录，dest也必须是一个目录。如果dest是不存在的路径，并且如果dest以/结尾或者src是目录，则dest被创建。如果src和dest是文件，如果dest的父目录不存在，任务将失败
Group:设置文件/目录的所属组
Mode:设置文件权限，模式实际上是八进制数字（如0644），少了前面的零可能会有意想不到的结果。从版本1.8开始，可以将模式指定为符号模式（例如u+rwx或u=rw,g=r,o=r）
Owner:设置文件/目录的所属用户
remote_src: yes 表示源文件位于远程主机上
```

### file模块（删除、创建、修改目录）

```
Path:必须参数，用于指定要操作的文件或目录
owner:用于指定被操作文件的属主，属主对应的用户必须在远程主机中存在，否则会报错。
group:用于指定被操作文件的属组，属组对应的组必须在远程主机中存在，否则会报错。
state: 参数为directory时，”directory”为目录之意，state的值设置为touch，则是文件的意思，创建软链接文件时，需将state设置为link。想要创建硬链接文件时，需要将state设置为hard，想要删除一个文件时state的值设置为absent。
src :当state设置为link或者hard时，表示我们想要创建一个软链或者硬链，所以，我们必须指明软链或硬链链接的哪个文件，通过src参数即可指定链接源。
recurse:当要操作的文件为目录，将recurse设置为yes，可以递归的修改目录中文件的属性。
```

### systemd模块(systemctl 命令)

```
enabled: yes 开启开机自启
name: 必须参数， 服务名称
state: started 启动服务 stopped 停止服务。如果服务已经停止，则不执行任何操作，restarted：重启服务。如果服务未在运行，则启动服务。 reloaded：重新加载服务配置。如果服务未在运行，则不启动服务。status：检查服务状态。不执行启动、停止、重启或重新加载操作。
```



### lineinfile模块（用于文件内的内容处理）

```
path:必须参数，指定要操作的文件
line:使用此参数指定文本内容
create:当要操作的文件并不存在时，是否创建对应的文件。
regexp:
```

### Replace模块（基于正则进行匹配和替换）

```
path:必须参数，指定要操作的文件
regexp:必须参数，在文件内容中查找的正则表达式
replace:要替换regexp的字符串匹配
```

### Setup模块（收集主机信息）

```
filter:用于进行条件过滤。如果设置，仅返回匹配过滤条件的信息。
```

###  Get_url 模块

```
url: 下载文件的url，支持http，https或FTP协议
dest: 下载到目标路径（绝对路径），如果目标是一个目录，就用服务器上面文件的名称，如果目标设置了命令，就用目标设置的名称
force：如果yes，dest不是目录，将每次下载文件，如果内容改变，替换文件。则只有在目标不存在时才会下载该文件
timeout: URL请求的超时时间,秒为单位
```

### template模块

```
src: 模板的文件位置  #Jinja2格式化模板的文件位置,必须字段
dest:  模板在远程主机上渲染的路径 #必须字段
mode: #文件的权限模式
```

## playbook 剧本

- hosts：执行的远程主句列表
- tasks：任务列表
- variables：变量
- Templates模板：可替换模板文件中的变量并实现一些简单逻辑的文件。
- Handlers和notify结合使用，由特定条件触发的操作，满足条件方才执行，否则不执行。
- tags标签：指定某条任务执行，用于选择执行playbook中的部分代码。ansible具有幂等性，因此会自动跳过没有变化的部分。

```
- hosts:              #指定对那些主机执行
  remote_user: root   #指定执行的用户
  gather_facts: false # Gathering Facts 收集各主机的 facts 信息，以方便我们在 paybook 中直接引用 facts 里的信息。
如果不需要用到 facts 信息的话，可以设置 gather_facts: false，来省去 facts 采集这一步以提高 playbook 效率。
  tasks: 
    - name: 
      
```

ansible-playbook

```
-v    #详细信息
-i	  #指定inventory 
-C	  #测试yaml文件语法是否正确
-t    #指定标签，只执行标签的tasks
--list-tags  #列出所有标签
```

### handlers和notify（触发器）

**用于当关注的资源发生变化时采取一定的操作。**

`Handlers` 是一组特殊的任务，它们只有在特定条件下才会被执行。通常，这些条件是由 `notify` 语句触发的。当一个 `notify` 语句被触发时，它会通知一个或多个 `handlers` 任务，告诉它们可以执行了。`Handlers` 通常用于在特定情况下执行一些不常见的操作，例如重启服务或重新加载配置文件。

`Notify` 是用于触发 `handlers` 的语句。当一个 `notify` 语句被执行时，它会通知一个或多个 `handlers` 任务，告诉它们可以执行了。通常，`notify` 语句会与某个任务关联，当这个任务执行成功时，`notify` 语句就会被触发。

```
- hosts: 192.168.56.11
  remote_user: root
  tasks:
    - name: install httpd package
      yum: name=httpd state=latest
    - name: install config file for httpd
      copy: src=/etc/httpd/conf/httpd.conf dest=/etc/httpd/conf/httpd.conf
      notify: restart httpd service
    - name: start httpd service
      service: enabled=true name=httpd state=started

  handlers:
    - name: restart httpd service
      service: enabled=true name=httpd state=restarted
```



### tags(标签)

tags用于让用户选择运行或跳过playbook中的部分代码，ansible具有幂等性，因此会自动跳过没有变化的部分，即便如此，有些代码为测试其确实没有发生变化的时候依然会比较长，此时确信其没有变化，就可以通过tags跳过这些代码。

```yaml
- hosts: webserver
  remote_user: root
  tasks:
    - name: copy
      copy: src=/root/a.sh dest=/opt
      tags: a
    - name: copy container
      copy: src=/root/containerd.sh dest=/opt
      tags: container
    - name: copy fun
      copy: src=/root/fun.sh dest=/opt
      tags: fun
```

```
 ansible-playbook  copy.yaml  --tags="container"  #使用了 --tags 标签后ansible只会运行 带有标签的 tasks
```

注意:如果你想同时运行多个标记，则可以使用逗号分隔多个标记

### 变量

变量名：只能由字母、数字和下划线组成，且只能以字母开头

 调用变量方式: 通过 “{{ var_name }}”,双引号加双大括号并且前后加空格调用。

变量的优先级：https://docs.ansible.com/ansible/2.9/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable

yaml文件中定义变量

```yaml
- hosts: 
  remote_user: root
  vars:
     - filename: test
   tasks:
     - name: touch file
       file: name=/opt/"{{ filename }}".log state=touch 
```

调用变量文件

```
vim var.yaml
filename: test

- hosts: 
  remote_user: root
  vars_files:
    - var.yaml  #可以写绝对路径，也可以写相对路径。
   tasks:
     - name: touch file
       file: name=/opt/"{{ filename }}".log state=touch
```

### facts变量

使用setup模块中变量

```
ansible  127.0.0.1  -m setup #获取变量名
```

默认会自动收集被控主机的环境信息，如果备考主机过多会很卡，如果并未使用请关闭

```
- hosts:
  gather_facts: false #关闭
```

优化facts

1 gather_subset：指定在执行主机信息收集（facts gathering）时应该收集哪些子集（subset）的信息。

`gather_subset` 选项可以用来控制收集哪些子集的信息。默认情况下，Ansible会尽可能地收集所有可用的信息。但在某些情况下，可能只需要收集特定的子集信息，以提高执行速度或减少收集的数据量。

`gather_subset` 可以设置为以下值之一：

- `all`：收集所有可用的信息子集。
- `hardware`：仅收集硬件相关的信息，例如CPU、内存、硬盘等。
- `network`：仅收集网络相关的信息，例如IP地址、网络接口等。
- `virtual`：仅收集虚拟化相关的信息，例如虚拟机类型、宿主机信息等。
- `ohai`：仅收集由Ohai插件提供的信息（仅限Linux主机）。
- 自定义子集：可以指定自定义的子集名称，以收集特定的信息。

```
- name: Gather facts
  hosts: all
  gather_facts: yes
  gather_subset: hardware
```

2 开启reids缓存



### template 模板

它可以通过 Jinja2 模板引擎来生成配置文件、脚本和其他文本文件。使用模板可以使 Ansible 在不同的主机上生成不同的文本内容，同时避免重复的手动修改。

template功能：可以根据和参考模块文件，动态生成相类似的配置文件

template文件必须存放于templates目录下，且命名为.j2结尾

yaml/yml 文件需和templates 目录平级，目录结构如下：

```
temnginx.yaml
templates
    nginx.config.j2
```



使用template模块

```
- hosts: webserver
  remote_user: root
  tasks:
    - name: template nginx config
      template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
```

### args指令

为模块提供额外的参数。它有以下几个用途:

1. 当模块的参数过多时,可以使用 args 将部分参数传递给模块,简化 playbook 的书写。
2. 当模块的参数名称和 playbook 中的其他变量冲突时,可以使用 args 避免冲突。
3. 当模块经常改动参数时,使用 args 可以减少 playbook 的修改。

### register 指令 

注册某个任务的输出结果,并在后续任务中使用。

```
- name: Run command and register result
  shell: /bin/echo "Hello world"
  register: hello_world_result
```

后续就可以 通过"{{ hello_world_result }}" 调用变量

# delegate_to(委派)

通过"`delegate_to`", 用户可以把某一个任务放在委托的机器上执行

```
- name: add host record 
  shell: 'echo "192.168.1.100 test.xyz.com" >> /etc/hosts'

 - name: add host record to center server 
  shell: 'echo "192.168.1.100 test.xyz.com " >> /etc/hosts'
  delegate_to: 192.168.1.1
```

第二个任务会在本机执行 而不是在远程主机执行

## jinj2

官方文档：https://docs.jinkan.org/docs/jinja2/

1、引用变量

```
{{ name }}
```

2、条件判断

if 语法

```
{% if  value  %}
  commmand
{% elif value2 %}
  command2
{% else %}
  command3
{% endif %}
```

for 循环语法（迭代列表）

```
{% for i in list %}
   command
{% endfor %}
```

注释：您可以使用 `{# ... #}` 语法在模板中添加注释。

```
{# 注释内容 #}
```

基本运维符：

| 运算符  |                             说明                             |
| :-----: | :----------------------------------------------------------: |
| +,-,*,/ | 加减乘除，+操作符也可用于字符串串联，*也可以用作重复字符串，“a” * 5 会得到5个a |

比较操作符：

| 比较符 |   说明   |
| :----: | :------: |
|   >    |   大于   |
|   <    |   小于   |
|   >=   | 大于等于 |
|   <=   | 小于等于 |
|   ==   |   等于   |
|   !=   |  不等于  |

其他操作符：

| 操作符 |                             说明                             |
| :----: | :----------------------------------------------------------: |
|   in   |                    测试 是否在列表字典中                     |
|   is   | is 可以配合函数 做很多测试操作，比如是否是数值，是否是字符串等 |

内置的测试函数：

|   函数名   |                          说明                           |
| :--------: | :-----------------------------------------------------: |
| `defined`  |              如果值已经定义，则返回 True。              |
|   `none`   |              如果值是 None，则返回 True。               |
| `sequence` | 如果值是序列（例如列表、元组、字符串等），则返回 True。 |
| `mapping`  |         如果值是映射（例如字典），则返回 True。         |
|  `string`  |              如果值是字符串，则返回 True。              |
|  `number`  |               如果值是数字，则返回 True。               |
| `integer`  |               如果值是整数，则返回 True。               |
|  `float`   |              如果值是浮点数，则返回 True。              |
| `boolean`  |              如果值是布尔值，则返回 True。              |
| `callable` |           如果值是可调用的对象，则返回 True。           |
|  `sameas`  |            如果值等于给定的值，则返回 True。            |

## playbook 中的判断

  when 语句可以实现条件测试，如果需要根据变量、facts或此前任务的执行结果来做为某task执行与否 的前提时要用到条件测试,通过在task后添加when子句即可使用条件测试，jinja2的语法格式

示例：

```yaml
- name: playbook when
  hosts: all
  tasks: 
    - name: taks1
      shell: systemctl stop firewalld
      when: ansible_distribution_major_version == "7"
```

可以用 `逻辑运算符` 来组合条件 and（并且，两个都要满足）和 or（或，满足一个条件即可）

```yaml
when: ansible_facts['distribution'] == "CentOS" and ansible_facts['distribution_major_version'] == "6"
```

```yaml
when: ansible_facts['distribution'] == "CentOS" or ansible_facts['distribution'] == "Debian"
```

如果您有多个条件，可以用括号对它们进行分组：

```yaml
when: (ansible_facts['distribution'] == "CentOS" and ansible_facts['distribution_major_version'] == "6") or (ansible_facts['distribution'] == "Debian" and ansible_facts['distribution_major_version'] == "7")
```



## playbook 使用循环迭代（with_items）(loop)

迭代：当有需要重复性执行的任务时，可以使用迭代机制 

对迭代项的引用，固定内置变量名为"item" 

要在task中使用with_items给定要迭代的元素列表 

注意: ansible2.5版本后,可以用loop代替with_items

```
- name: Add several users
  hosts:
  tasks:
    - name:
      user: name: "{{ item.name }}" state: present groups: "{{ item.groups }}"
  loop:
    - { name: 'testuser1', groups: 'wheel' }
    - { name: 'testuser2', groups: 'root' }
```

### 嵌套循环

```
- hosts: 192.168.56.11
  remote_user: root
  tasks:
    - name: give users access to multiple databases
      command: "echo name={{ item[0] }} priv={{ item[1] }} test={{ item[2] }}"
      with_nested:
        - [ 'alice', 'bob' ]
        - [ 'clientdb', 'employeedb', 'providerdb' ]
        - [ '1', '2', ]
```

### 遍历字典

```
- name: Using dict2items
  ansible.builtin.debug:
    msg: "{{ item.key }} - {{ item.value }}"
  loop: "{{ tag_data | dict2items }}"
  vars:
    tag_data:
      Environment: dev
      Application: payment
```

## roles角色

角色允许您根据已知的文件结构自动加载相关的变量、文件、任务、处理程序和其他 Ansible 工件。按角色对内容进行分组后，您可以轻松地重复使用它们并与其他用户共享它们。

```
roles目录结构：
roles:  
 project:   #定义项目名称比如 nginx、mysql、tomcat
   tasks:   #任务playbook目录 至少包含一个名为main.yml的文件，其定义了此角色的任务列表，此文件可以使用include包含其他位于此目录的task文件。
   files:  #存放于copy或scripts等模块调用的文件
   vars:  #应当包含一个main.yml文件，用于定义此角色用到的变量
   templates: #模板资源文件
   handlers:  #触发器文件
   default:  #为当前角色设定默认变量时使用此目录，应当包含一个main.yml文件，默认低优先级变量
   meta:  #应当包含一个main.yml文件，用于定义此角色的特殊设定及其依赖关系
```

使用 import_tasks: 来替代 - include：

```
# roles/example/tasks/main.yml
- name: Install the correct web server for RHEL
  import_tasks: redhat.yml
  when: ansible_facts['os_family']|lower == 'redhat'

- name: Install the correct web server for Debian
  import_tasks: debian.yml
  when: ansible_facts['os_family']|lower == 'debian'
```

