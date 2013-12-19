.. _helloworld:

Hello World
=============


作者背景
---------

作者是一个使用多种语言开发复杂程序并且拥有十多年经验的软件工程师。作者第一次学习 Python 是在为一个 C++ 库创建绑定的时候。

除了 Python，作者曾经用 PHP, Ruby, Smalltalk 甚至 C++ 写过 web 应用。在所有这些中，Python/Flask 组合是作者认为最为自由的一种。

应用程序简介
-------------

作为本教程的一部分--我要开发的应用程序是一个极具特色的微博服务器，我称之为 *microblog* 。

我会随着应用程序的不断地进展将涉及到如下这些主题：

* 用户管理，包括管理登录，会话，用户角色，权限以及用户头像。
* 数据库管理，包括迁移处理。
* Web 表单支持，包括对各个字段的验证。
* 分页处理。
* 全文搜索。
* 用户邮件提醒。
* HTML 模板。
* 支持多国语言。
* 缓存以及其它性能优化技术。
* 开发以及生产服务器的调试技巧。
* 在生产服务器上安装。

我希望这个应用程序将能够成为编写其它类型的 web 应用程序的一个样板，当它完成的时候。

要求
-----

如果你有一台能够运行 Python 2.6 或者 Python 2.7 的机器，可能你将会很轻松。该教程中的应用程序能够完美地运行在 Windows, OS X 以及 Linux 上。

本教程假定你很熟悉操作系统的终端窗口(命令提示符为 Windows 用户)，清楚基本命令行文件管理功能。如果你还不熟悉这些的话，我强烈建议你先学习使用命令行，比如如何创建文件夹等，接着再继续。

最后，你应该还能够很舒服地(熟练地)编写 Python 代码。强烈推荐熟悉 Python 的 `Python 模块和包 <http://docs.python.org/tutorial/modules.html>`_ 。

安装 Flask
------------

好的，让我们开始吧！

现在我们必须开始安装 Flask 以及一些我们会用到的扩展。我首选的方式就是创建一个 `虚拟环境 <http://pypi.python.org/pypi/virtualenv>`_ ,这个环境能够安装所有的东西，而你的主 Python 不会受到影响。另外一个好处就是这种方式不需要你拥有 root 权限。

因此，打开一个终端窗口，选择一个你想要放置应用程序的位置以及创建一个包含它的新的文件夹。让我们把这个应用程序的文件夹称为 **microblog** 。

接下来，下载 `virtualenv.py <https://raw.github.com/pypa/virtualenv/1.9.X/virtualenv.py>`_ 并且把它置于新创建的文件夹中。

为了创建一个虚拟环境，请输入如下的命令行 ::

	python virtualenv.py flask

上面的命令行在 *flask* 文件夹中创建一个完整的 Python 环境。

虚拟环境是能够激活以及停用的，如果需要的话，一个激活的环境可以把它的 *bin* 文件夹加入到系统路径。我个人是不喜欢这种特色，所以我从来不激活任何环境相反我会直接输入我想要调用的解释器的路径。

如果你是在 Linux, OS X 或者 Cygwin 上，通过一个接一个输入如下的命令行来安装 flask 以及扩展::

	flask/bin/pip install flask==0.9
	flask/bin/pip install flask-login
	flask/bin/pip install flask-openid
	flask/bin/pip install flask-mail
	flask/bin/pip install sqlalchemy==0.7.9
	flask/bin/pip install flask-sqlalchemy==0.16
	flask/bin/pip install sqlalchemy-migrate
	flask/bin/pip install flask-whooshalchemy==0.54a
	flask/bin/pip install flask-wtf
	flask/bin/pip install pytz==2013b
	flask/bin/pip install flask-babel==0.8
	flask/bin/pip install flup

如果是在 Windows 上的话，命令行有些不同 ::

	flask\Scripts\pip install flask==0.9
	flask\Scripts\pip install flask-login
	flask\Scripts\pip install flask-openid
	flask\Scripts\pip install sqlalchemy==0.7.9
	flask\Scripts\pip install flask-sqlalchemy==0.16
	flask\Scripts\pip install sqlalchemy-migrate
	flask\Scripts\pip install flask-whooshalchemy==0.54a
	flask\Scripts\pip install flask-wtf
	flask\Scripts\pip install pytz==2013b
	flask\Scripts\pip install flask-babel==0.8
	flask\Scripts\pip install flup

这些命令行将会下载以及安装我们将会在我们的应用程序中使用的所有的包。

注意的是我们使用的是 Flask 0.9，这不是最新的版本。Falsk 0.10 刚发布不久因此很多扩展暂时没有更新，在扩展包和最新版本的 Flask 之间存在一些不兼容性。

Windows 上的用户需要多做一步。如果你有良好的观察能力的话，你可能注意到，*flask-mail* 不在 Windows安装命令行中。这个扩展在 Windows 安装的不干净，因此我们需要通过一个变通的方式来安装::

	flask\Scripts\pip install --no-deps lamson chardet flask-mail

这里我不会去讨论细节，如果你对此感兴趣的话可以参看 flask-mail 文档。

如果所有的包成功地安装，你可以删除 *virtualenv.py* ，因为我们不再需要它了。

在 Flask 中的 "Hello, World"
------------------------------

现在在你的 *microblog* 文件夹中下有一个 *flask* 子文件夹，这里有 Python 解释器以及 Flask 框架以及我们将要在这个应用程序中使用的扩展。 是时候去编写我们第一个 web 应用程序！

在 *cd* 到 *microblog* 文件夹后，我们开始为应用程序创建基本的文件结构::
	
	mkdir app
	mkdir app/static
	mkdir app/templates
	mkdir tmp


我们的应用程序包是放置于 *app* 文件夹中。子文件夹 *static* 是我们存放静态文件像图片，JS文件以及样式文件。子文件夹 *templates* 显然是存放模板文件。

让我们开始为我们的 *app* 包(文件 *app/__init__.py* )创建一个简单的初始化脚本::

	from flask import Flask

	app = Flask(__name__)
	from app import views

上面的脚本简单地创建应用对象，接着导入视图模块，该模块我们暂未编写。

视图是响应来自网页浏览器的请求的处理器。在 Flask 中，视图是编写成 Python 函数。每一个视图函数是映射到一个或多个请求的 URL。

让我们编写第一个视图函数(文件 *app/views.py* )::

	from app import app

	@app.route('/')
	@app.route('/index')
	def index():
    	return "Hello, World!"

其实这个视图是非常简单，它只是返回一个字符串，在客户端的网页浏览器上显示。两个 *route* 装饰器创建了从网址 */* 以及 */index* 到这个函数的映射。

能够完整工作的 Web 应用程序的最后一步是创建一个脚本，启动我们的应用程序的开发 Web 服务器。让我们称这个脚本为 *run.py*，并把它置于根目录::

	#!flask/bin/python
	from app import app
	app.run(debug = True)

这个脚本简单地从我们的 app 包中导入 *app* 变量并且调用它的 *run* 方法来启动服务器。请记住 *app* 变量中含有我们在之前创建的 *Flask* 实例。

要启动应用程序，您只需运行此脚本（*run.py*）。在OS X，Linux 和 Cygwin 上，你必须明确这是一个可执行文件，然后你可以运行它::

	chmod a+x run.py

然后脚本可以简单地按如下方式执行::

	./run.py

在 Windows 上过程可能有些不同。不再需要指明文件是否可执行。相反你必须运行该脚本作为 Python 解释器的一个参数::

	flask/Scripts/python run.py

在服务器初始化后，它将会监听 5000 端口等待着连接。现在打开你的网页浏览器输入如下 URL::

	http://localhost:5000

另外你也可以使用这个 URL::

	http://localhost:5000/index

你看清楚了路由映射是如何工作的吗？第一个 URL 映射到 */*，而第二个 URL 映射到 */index*。这两个路由都关联到我们的视图函数，因此它们的作用是一样的。如果你输入其它的网址，你将会获得一个错误，因为只有这两个 URL 映射到视图函数。

你可以通过 *Ctrl-C* 来终止服务器。到这里，我将会结束这一章的内容。对于不想输入代码的用户，你可以到这里下载代码：`microblog-0.1.zip <https://github.com/miguelgrinberg/microblog/archive/v0.1.zip>`_。


下一步？
--------

下一章我们将会小小修改下我们的应用，使用 HTML 模板。我希望在下一章再见到大家！




