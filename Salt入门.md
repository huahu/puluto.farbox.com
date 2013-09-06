什么是Saltstack？
=================

以Apache2 协议开源的管理服务器基础设施以及服务的轻量级工具（所谓轻量级是说工具结构简单、并不是说功能简单）
-----------------
可以远程执行命令管理，也可以进行服务器相关的状态管理，通常我们认为salt是fedora func与 puppet 的结合体

使用Python编写，通过证书加密保证通讯安全，使用0MQ作为通讯承载方式具有很高的性能以及灵活性

标准化模块实现各种功能，没有自己开发不通用的技术
-----------------
* Python 2.6 >= 2.6 <3.0
* ZeroMQ >= 2.1.9
* pyzmq >= 2.1.9 - ZeroMQ Python bindings
* PyCrypto - The Python cryptography toolkit
* msgpack-python - High-performance message interchange format
* YAML - Python YAML bindings
* Jinja2 - parsing Salt States (configurable in the master settings)
* Python-m2crypto - M2Crypto Python bindings

高效率：
------------------
    Salt的C/S核心采用0MQ消息队列以及msgpack进行通信，网络开销非常小并且可以并行异步执行任务，所以单台普通配置master可以支撑
    几百台甚至上千台的minion管理，并且速度能得到保障。

可扩展：
------------------
    Salt的部署扩展非常灵活，自带分布式部署方式，通过salt-syndic服务进行分布式管理以及0.16版本新增的多master结构，
    功能的扩展上可以很容易的编写自定义的模块进行拓展，并且扩展模块很容易利用现有的模块进行功能的强化。

灵活性：
------------------
    Salt在使用方面对目标端的匹配可以使用多种方式，可以通过指定分组、linux通配符、正则表达式、grains模块信息进行目标端的过滤，
    salt本身提供了丰富的返回信息输出方式，除了可以终端输出还可以指定返回信息输出到各种数据存储服务当中，比如mysql、redis、
    mangodb等并且可以很方便的根据自己的需要对returnner模块进行扩展满足自己的特殊需求。

安装Saltstack
=============

支持的系统：
-------------
已经在常见的系统打包好了，可以直接使用包管理工具安装使用
* Fedora
* RedHat Enterprise Linux / Centos (EPEL 5, EPEL 6)
* Ubuntu (PPA)
* Arch (AUR)
* FreeBSD
* Gentoo
* Debian

如果特殊环境不方便直接使用包管理工具可以源码安装：
------------------
* Python 2.6 >= 2.6 <3.0
* ZeroMQ >= 2.1.9
* pyzmq >= 2.1.9 - ZeroMQ Python bindings
* PyCrypto - The Python cryptography toolkit
* msgpack-python – 高性能消息序列化工具
* YAML - 一种通用的结构化数据格式、GAE的配置就是使用这个
* MarkupSafe – jinja2依赖
* Jinja2 - 默认模版模块，如需要支持mako模版也需要安装
* Python-m2crypto - M2Crypto Python bindings

**Salt带来的惊喜是他支持Windows的操作，当然由于Windows本身的原因目前功能有限，不过正在完善中。**

Master:
-------------------
安装完成以后修改/etc/salt/master配置文件后执行::

    /etc/init.d/salt-master start

Tips: *默认可以不用进行任何配置便可以启动*

Minion:
-------------------
安装完成以后修改/etc/salt/minion配置文件里面的Master：行指定master服务器的位置::

    /etc/init.d/salt-minion start

配置文件里面所有的项冒号后面需要跟一个空格，这是yaml文件的语法,例如::

    Master: x.x.x.x

如果是编译安装在/etc/目录和init.d目录是没有相关文件的，需要进入源码目录执行::

    cp -a conf /etc/salt
    cp pkg/salt-master.service /etc/init.d/salt-master
    cp pkg/salt-minion.service /etc/init.d/salt-minion

安装启动完成后在Master机器上执行::

    salt-key –L             #可以列出所有minion发送到Master的认证key信息
    salt-key –A             #可以通过所有minion的认证
    salt-key –a  minion-id  #通过指定id的minion认证，minion-id默认是机器名，
                            #也可以通过minion的配置文件里id一项进行指定，不同minion的id不能相同

认证完成后就可以执行salt命令进行管理了，测试::

    salt '*' test.ping


Salt的使用
============================
salt包含多个子系统，其中比较主要的有:

* modules - salt中最基础的系统，主要功能为批量发送指令控制服务器，使用模块封装了很多平时常用的功能
* states - 与puppet的配置管理类似的功能，优点是简单灵活容易扩展，成熟度正在向puppet靠拢
* runner - 用于在服务端定义一些复杂的功能模块分发到minion执行，用到的机会比较少

常用的与主要子系统配合的辅助子系统:

* target - 主要工作是匹配过滤minion，在对服务器执行主要功能的时候对客户端进行筛选精确匹配
* grains - 存储minion端系统相关和部分硬件信息，里面的信息可以用于资产管理，因为默认会统计配置信息以及硬件序列号
* render - 用于salt中模版文件的渲染，各个系统的配置文件均支持模版语言，默认支持jinja2，可以扩展支持mako等模版引擎
* pillar - 用于对minion端敏感信息的分发以及作为一个统一的变量系统，里面设定的变量或者内容只对其匹配到的客户端进行精确分发
* schedule - 用于定时在Master或者minion端执行特定的任务的模块，相当于Linux系统的crontab，这个更便于管理
* returner - 用于特定方式返回salt任务执行后的结果，可以将结果输出到不同的存储端，比如mysql、redis、syslog、local、mangodb等
* acl system -

modules的使用
-----------------------------------------
modules服务是salt远程执行命令的服务，是salt中最基础的功能，salt很多扩展功能均利用modules
modules的模块中还有专门针对虚拟化和云计算的模块，包括非常详细的virt模块和虚拟网卡模块以及ec2的模块：

**以下为modules服务的一些实例**
安装完成后测试所有通过认证的服务器，并且id为test1机器安装vim::

    salt '*' test.ping
    salt 'test1' pkg.install vim

Tips: *pkg模块的参数中，包名是依赖系统所确定的包名，不同发行版可能会出现不同，比如apache在redhat系列叫httpd，debian中叫apache2*

modules命令使用格式::

    salt 'target' modules.function arg1 arg2 ... args_key=value

salt的modules已经自带了很多常用模块，可以从官方文档看到，如果没有相应的模块也可以使用cmd.run模块通过shell实现简单的功能::

    salt '*' cmd.run 'uptime'
    salt terminal2 cmd.run "cat /etc/hosts"
    >
    terminal2:
      #		Do not remove the following line, or various programs
      #		that require network functionality will fail.
      127.0.0.1		localhost
      ...

modules服务中有很多模块，如果对模块的用法以及函数难以记录可以使用sys.argspec模块对其他模块的用法进行查询::

    salt 'terminal2' sys.argspec saltutil
    >
    terminal2:
    ----------
      saltutil.find_job:
        ----------
        args:
            - jid
        defaults:
            None
        kwargs:
            None
        varargs:
            None
      saltutil.is_running:
        ...
      ...

Tips: *此命令可以输出saltutil模块的所有函数以及函数的参数*

如果需要对某个具体函数进行查询可以和上面的用法相同::

    salt 'terminal2' sys.argspec saltutil.is_running
    >
    terminal2:
    ----------
      saltutil.is_running:
        ----------
        args:
            - fun
        defaults:
            None
        kwargs:
            None
        varargs:
            None

也可以使用sys.doc命令查询模块的相关文档::

    salt 'terminal2' sys.doc saltutil.is_running
    >
    saltutil.is_running:

      If the named function is running return the data associated with it/them.
      The argument can be a glob

      CLI Example:

        salt '*' saltutil.is_running state.highstate

Tips: *直接运行salt 'terminal2' sys.doc可以输出当前可用的所有模块信息*



state服务的使用
-----------------------------------------
配置state功能的描述文件根目录，state描述文件使用yaml格式以.sls为后缀，在指定的根目录以top.sls为入口文件进行解析执行::

    vim  /etc/salt/master

    file_roots:
    base:
      - /data/srv/salt/

修改好配置文件，重启Master服务::

    cd /data/srv/salt/

    vim top.sls      #输入以下内容
    base:            #环境名称，file_roots可以指定多个环境，不同环境不同目录
      '*':           #需要匹配的客户端，*号代表所有minion
        - test       #test代表匹配到的客户端需要执行的任务或者模块

建立test任务，任务可以是一个sls文件，也可以是目录::

    vim test.sls    #如果test任务是一个目录，那么如果直接调用test任务需要把任
                    #务内容写到test目录下的init.sls文件中
                    #在test目录中非init.sls文件名的调用需要的格式为test.task，
                    #或者在init文件中include另外的文件内容
    zabbix_agentd:
      pkg:
        - installed

    /etc/zabbix/zabbox_agentd.conf:
      file:
        - managed
        - source: salt://file/zabbix_agentd.conf
        - mode: 644
        - user: root

    zabbix_agentd_service:
      service.running:
        - name: zabbix_agentd
        - enable: True
        - enable: reload
        - require:
          - pkg: zabbix_agentd
        - watch:
          - file: /etc/zabbix/zabbox_agentd.conf

书写state系统的sls文件的标准为::

    task_id:                         #定义一条命令的id，id在整个系统中是唯一的，不能重复
      module:                        #state支持的模块名称
        - function                   #模块的函数名称，你需要执行的具体任务
        - args_key: args_value       #模块需要指定的一些参数


target的使用
----------------------------------------
target是用于在执行modules或者state以及后面要讲到的pillar系统时匹配目标机器的功能
对于目标的匹配target支持多种模式:

Linux的shell通配符匹配，默认情况下target均使用此类模式::

    salt 'web*' test.ping                #匹配所有经过认证的minion客户端
    salt 'web[1-9x-z]' test.ping         #可以匹配到web1-9以及webx、weby、webz
    salt 'web?' test.ping                #匹配web后面只有一个字符的客户端

正则表达式匹配,使用正则匹配需要命令行指定`-E`参数::

    salt -E 'web1-(prd|dev)' test.ping

如果是在state的top.sls中使用正则匹配，需要在匹配表达式里指定`match`参数的值为pcre::

    base:
      'web1-(prd|dev)':
      - match: pcre
      - webserver

在执行对多个minion的操作时，客户端数量较小比较难使用通配符或者正则的情况下可以使用`-L`参数手动输入minion列表::

    salt -L 'web1,web2,web3' test.ping

通过Grains系统的值进行匹配，指定参数`-G`::

    salt -G 'os:CentOS' test.ping
    salt -G 'cpuarch:x86_64' test.ping

使用`-N` 参数可以使用预先设定的分组进行minion的匹配::

    salt -N group1 test.ping

设定group的方式为，修改Master的配置文件添加nodegroups段，并且进行分组的时候可以利用其他几类匹配方式帮助分组::

    nodegroups:
      group1: 'L@foo.domain.com,bar.domain.com,baz.domain.com or bl*.domain.com'
      group2: 'G@os:Debian and foo.domain.com'

和上面分组一样，执行命令或者state的时候同样可以进行混合匹配,命令需要指定`-C`参数表示混合匹配::

    salt -C 'webserv* and G@os:Debian or E@web-dc1-srv.*' test.ping

在state和pillar中使用的时候需要指定match参数的值为compound::

    base:
      'webserv* and G@os:Debian or E@web-dc1-srv.*':
      - match: compound
      - webserver


grains服务介绍
-----------------
Grains服务是salt里面记录一些系统重要信息的模块，并且这些信息可以方便的查看使用和自定义

查看系统里面所有的Grains项目::

  >
  salt 'terminal2' grains.ls

    terminal2:
    - biosreleasedate
    - biosversion
    - cpu_flags
    - cpu_model
    - cpuarch
    - defaultencoding
    - defaultlanguage
    - domain
    - fqdn
    - gpus
    - host
    - id
    - ip_interfaces
    - ipv4
    - kernel
    ...

查看系统Grains所有项目以及对应的值::

  >
  salt 'terminal2' grains.items

    terminal2:
      biosreleasedate: 09/08/2010
      biosversion: 1.4.8
      cpu_flags: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse
              sse2 ss ht tm syscall nx pdpe1gb rdtscp lm constant_tsc ida nonstop_tsc pni monitor ds_cpl vmx smx est
              tm2 cx16 xtpr popcnt lahf_lm
      cpu_model: Intel(R) Xeon(R) CPU           E5620  @ 2.40GHz
      cpuarch: x86_64
      defaultencoding: UTF8
      defaultlanguage: zh_CN
      domain:
      fqdn: terminal2
      gpus:
      host: terminal2
      id: terminal2
      ip_interfaces: {'sit0': [], 'lo': ['127.0.0.1'], 'eth1': ['10.11.10.21'], 'eth0': ['121.10.118.21',
      '112.91.18.21']}
      ipv4:
        127.0.0.1
        10.11.10.21
        121.10.118.21
        112.91.18.21
      kernel: Linux
      kernelrelease: 2.6.18-164.el5

查看某一个Grains项目的值::

  >
  salt 'terminal2' grains.item os

    terminal2:
      os: CentOS


render系统的介绍
-----------------------------------------
在Salt中模版系统是一个很重要的功能，可以帮助我们尽可能的少写代码以及增强salt应用的灵活性，render系统默认使用jinja2和yaml，可以
通过扩展或者设置支持json、mako、py、pydsl、wempy、stateconf。

在render系统中包括两类文件表述的支持:

* 一类是传统意义的模版，即jinja2、mako，这些模版系统包括逻辑处理支持以及一些简单的数据处理语句
* 另一类其实是一种数据格式，不支持逻辑语句，比如yaml、json、pydsl这些就属于数据格式

jinja2在state服务中的使用::

    zabbix_agentd:
      pkg:
        - installed
  
    /etc/zabbix/zabbox_agentd.conf:
      file:
        - managed
        #通过Grains里面的os信息判断系统类型，分发不同的文件，也可以在任何地方使用jinja2模版的其他功能
        {% if grains['os'] == 'CentOS' %}
        - source: salt://file/zabbix_agentd.conf
        {% else %}
        - source: salt://file/zabbix_agentd_other.conf
        {% endif %}
        - mode: 644
        - template: jinja
        - user: root

通过设定上面statefile的template参数为jinja，也可以使state管理的文件能使用jinja2作为模版进行渲染::

  vim /srv/salt/prd/source/conf/zabbix_agentd.conf

    ### Option: Server
    #       List of comma delimited IP addresses (or hostnames) of Zabbix servers.
    #       Incoming connections will be accepted only from the hosts listed here.
    #       No spaces allowed.
    #       If IPv6 support is enabled then '127.0.0.1', '::127.0.0.1', '::ffff:127.0.0.1' are treated equally.
    #
    # Mandatory: no
    # Default:
    # Server=
    {% if grains['os'] == 'CentOS' %}
    Server=10.11.10.21
    {% else %}
    Server=10.12.10.130
    {% endif %}
    ...

*我们可以通过自定义grains来为模版加入很大的灵活性，并且模版系统还可以引用pillars系统的元素，这个后面会讲到*


pillar服务的使用
-----------------------------------
Pillar服务是salt用于将一些全局数据分发到minion的一个接口，pillar以salt state类似的方式组织数据，存放与pillar的数据只会分发
到定义里所匹配到的minion客户端，所以可以用于敏感信息的分发，比如用户名密码之类的安全数据。
但目前我们一般将其作为一个全局的变量系统使用，pillar的数据可以供state和render服务所使用。

在Master的配置文件里设定pillar服务::

  vim /etc/salt/master

    pillars_roots:
      base:
        - /srv/pillar/

与state系统一样，使用pillar的方式是先定义top.sls::

  vim /srv/pillar/top.sls

    base:                 #默认环境
      '*':                #匹配的目标
        - system          #引用定义的system文件或者模块
        - software        #引用定义的software文件或者模块

定义了top.sls以后，需要top里面所引用的文件或者模块，这里以文件为例::

*默认pillar的数据使用K、V方式存储，V可以是字符或者列表，并且下面例子也使用了render服务*

  vim /srv/pillar/system.sls

    {% if grains['os_family'] == 'Suse' %}
    vim: 'vim'
    vimrc: 'vimrc_suse'
    nrpe_init: 'init-script.suse.in'
    {% elif grains['os_family'] == 'RedHat' %}
    vim: 'vim-enhanced'
    vimrc: 'vimrc_redhat'
    nrpe_init: 'init-script.in'
    {% elif grains['os_family'] == 'Debian' %}
    vim: 'vim'
    vimrc: 'vimrc_debian'
    nrpe_init: 'init-script.debian.in'
    {% endif %}

定义完成后查看pillar的数据::

  >
  salt 'terminal2' pillar.data

    terminal2:
      build_root:
        /usr/local
      nrpe_init:
        init-script.in
      salt_src:
        salt://source
      salt_tmp:
        /tmp/salt
      sys:
        ----------
        profile:
          profile_redhat
      vim:
        vim-enhanced
      vimrc:
        vimrc_redhat
      ....

在state中调用pillar的数据::

*在state中设定安装vim并且同步配置文件，因为不同系统vim包名不同，所以我们把这些差异都放到前面的pillar里面*
*这样在编辑statefile的时候就只需要调用变量，他会根据不同的系统给出不同的包名*
*在state所管理的文件里面也可以引用pillar的数据，所以我们可以把一些软件的特定配置写到pillar服务*

  vim /srv/salt/prd/states/vim/init.sls

    vimrc:
      pkg.installed:
        - name: {{ pillar['vim'] }}
      file.managed:
        - name: /etc/vimrc
        - source: {{ pillar['salt_src'] }}/conf/{{ pillar['vimrc'] }}
        - mode: 644
        - require:
          - pkg: {{ pillar['vim'] }}

在pillar中的数据也可以划分层级逐层调用,下面设定一个多级的pillar定义文件::

  vim /srv/pillar/system/init.sls

    sys:
    {% if "RedHat" == grains['os_family'] %}
      profile: profile_redhat
    {% elif "Suse" == grains['os_family'] %}
      profile: profile_suse
    {% elif "Debian" == grains['os_family'] %}
      profile: profile_debian
    {% endif %}

在使用上面定义的数据时就需要根据定义时的层级进行访问，可以方便的为数据分类::

  #其中profile的文件名就经过了两层，我们可以在定义软件的配置时把第一层命名为软件名，第二层定义软件的配置可以方便管理
  vim /srv/salt/prd/states/users/init.sls

  /etc/profile:
    file:
      - managed
      - source: {{ pillar['salt_src'] }}/conf/{{ pillar['sys']['profile'] }}
      - mode: 644
      - user: root


Schedule系统的使用
--------------------
schedule系统用于指定时间循环执行Master或者minion上的任务，当我们写好statefile需要保持minion处于指定状态的时候，就可以在
指定时间间隔执行state.highstate模块来保证state的同步。

schedule系统的启用方式::

#编辑minion的配置文件加入以下配置，重启salt-minion端
  schedule:                         #schedule系统的标识，不能更改
    highstate:                      #需要执行的任务ID，可以任意更改不能重复，建议使用函数的名称表示
      function: state.highstate     #function后面跟 `模块.函数名`表示需要执行的任务
      minutes: 60                   #表示每60分钟执行一次
*时间周期还可以指定minutes和hours，指定时间的所有项的总时间加在一起表示一个周期*
*根据以上格式，schedule里面可以加入任何想执行的模块，这些模块就是salt的modules系统里面包含的模块，也可以进行自定义模块*

schedule系统可以和returner结合将任务执行结果输出到指定位置::

#表示每60秒统计一次系统的uptime，并将执行结果存入mysql数据库
  schedule:
    uptime:
      function: status.uptime
      seconds: 60
      returner: mysql
*returner系统的详细介绍将在后续完善，目前还没有使用returner系统*


**本教程为基础教程，由于我比较糟糕的表述能力，请各位看官见谅**

**更多教程请参见**
__ http://saltstack.cn



