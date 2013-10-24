Celery功能简介
======================

Celery(芹菜)是一个异步任务队列/基于分布式消息传递的作业队列。

Celery用于生产系统每天处理数以百万计的任务。

Celery是用Python编写的，但该协议可以在任何语言实现。它也可以与其他语言通过webhooks实现。

由于Celery 3.0系列对以前的系列进行了大量重构优化，现在开始使用就没必要研究旧版本了，所以此介绍以3.0.24的文档为基础。

Celery的工作结构
----------------------
在使用Celery的时候要明白它的大致结构，Celery的结构非常简单，大致分为3个部分：

1. worker部分负责任务的处理，即工作线程，在我的理解中工作线程就是你写的python代码，当然还包括python调用系统工具功能
2. broker部分负责任务消息的分发以及任务结果的存储，这部分任务主要由中间数据存储系统完成，比如消息队列服务器RabbitMQ、redis、
   Amazon SQS、MongoDB、IronMQ等或者关系型数据库，使用关系型数据库依赖sqlalchemy或者django的ORM
3. Celery主类，进行任务最开始的指派与执行控制，他可以是单独的python脚本，也可以和其他程序结合，应用到django或者flask等
   web框架里面以及你能想到的任何应用

Celery的安装
----------------------
Celery只是一个python包，所以可以通过pip或者easy_install安装

pip install celery
easy_install install celery

除此之外还需要安装broker的系统，我使用的是redis，除了安装redis以外还需要安装celery-with-redis
pip install celery-with-redis
使用其他类型的broker请参见官方文档：
__ http://docs.celeryproject.org/en/latest/getting-started/first-steps-with-celery.html#using-a-database

Celery的初步使用
----------------------
启动celery之前先架设好broker服务，安装好redis后以默认方式启动就可以了。
连接方式为：redis://localhost:6379/0

接下来编写任务脚本tasks.py,这个脚本在worker部分和任务分发部分都需要用到：

    from celery import Celery

    celery = Celery('tasks', broker='redis://localhost:6379/0')

    @celery.task
    def add(x, y):
        return x + y

执行命令启动worker进行：

    #这个命令要在tasks.py文件目录运行，命令表示以worker模式启动一个名为tasks的APP
    #worker的名称是test-worker1，一台服务器上可以启动多个worker，只要名称不同，
    #启动好的worker会自动根据tasks.py的信息注册到broker服务中，等待分发任务。

    celery -A tasks worker --loglevel=info --hostname=test-worker1

执行任务,使用delay()放，如下：

    >>> from tasks import add
    >>> add.delay(4, 4)

也可以使用apply_async()方法，把结果存储在类似broker的backend系统中，可以和broker在同一个服务中，
更改tasks.py中的实例化celery一行,加入backend参数：
    celery = Celery('tasks', broker='redis://localhost:6379/0', backend='redis://localhost:6379/0')

重新执行任务，使用apply_async把结果存储下来，在需要的时候调用get()进行获取，如下：

    >>> from tasks import add
    >>> result = add.apply_async(4, 4)
    >>> result.get()

Celery配置
------------------------
Celery有很多全局变量，不配置的情况下取默认值，当我们需要配置的时候可以把所有的参数写到一个py文件中然后在
任务文件中进行加载，也可以直接用一个类写到任务文件中，还可以直接对celery类的conf对象直接进行update操作：

方法1，直接加载py文件：
    celeryconfig.py:

    BROKER_URL = 'amqp://'
    CELERY_RESULT_BACKEND = 'amqp://'
    CELERY_TASK_SERIALIZER = 'json'
    CELERY_RESULT_SERIALIZER = 'json'
    CELERY_TIMEZONE = 'Europe/Oslo'
    CELERY_ENABLE_UTC = True

    from celery import Celery

    celery = Celery()
    celery.config_from_object('celeryconfig')

方法2，直接对conf进行update：
    from celery import Celery

    celery = Celery()
    celery.conf.update(
    CELERY_TASK_SERIALIZER='json',
    CELERY_RESULT_SERIALIZER='json',
    CELERY_TIMEZONE='Europe/Oslo',
    CELERY_ENABLE_UTC=True,
    )

    or

    celery.conf.CELERY_TASK_SERIALIZER = 'json'

方法3，直接加载类或对象:
    from celery import Celery

    celery = Celery()

    class Config:
        CELERY_ENABLE_UTC = True
        CELERY_TIMEZONE = 'Europe/London'

    celery.config_from_object(Config)

Celery任务分发控制
-----------------------------
在celery里面任务分发控制叫task routing即任务路由

celery的分发控制使用比较简单，但是高级功能比较复杂，我还不能完全理解，就介绍一下最基础的任务路由方法。

在worker进程启动的时候可以使用参数-Q指定当前worker所能接受的队列消息：

    celery -A tasks.tasks worker --loglevel=info --hostname=testq-worker -Q 'testq'

然后在任务分发的过程中，调用apply_async或者delay方法中指定queue参数，当queue与worker的-Q相匹配时任务
就可以被分发到相应的worker进程中：

    >>> from tasks import add
    >>> result = add.apply_async(4, 4, queue='testq')
    >>> result.get()

更高级的使用方法请大家研究官网的文档：
__ http://docs.celeryproject.org/en/latest/userguide/routing.html


Celery的管理
-------------------

celery的管理有几种方式，比较直观的有一个叫flower的webui，可以提供任务查询，worker的生命管理以及路由管理，可以在界面上
进行实时的路由key添加（就是在worker启动时-Q参数指定的值）

使用方式为：

    #安装
    pip install flower

    #启动
    celery flower --port=5555 --broker=redis://localhost:6379/0

    访问localhost的5555端口即可以使用

还有一种对任务进行实时监控的方式为celery本身提供的events的功能，启动方式为：

    #启动一个字符界面下管理的工具，对任务进行方便的跟踪管理
    celery events --broker=redis://localhost:6379/0


结束语
-------------------
celery还有很多功能没来得及研究，我准备把celery应用于服务器管理中一些任务的执行，来代替linux的crontab和一些
手工操作，提升更强的灵活性以及更加直观
