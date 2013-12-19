.. _templates:

模板
======


回顾
-------

如果你依照 :ref:`helloworld` 这一章的话，你应当有一个完全工作的简单的 web 应用程序，它有着如下的文件结构::

	microblog\
      	flask\
        	<virtual environment files>
      	app\
        	static\
        	templates\
        	__init__.py
        	views.py
      	tmp\
      	run.py

你可以执行 *run.py* 来运行应用程序，接着在你的网页浏览器上打开 *http://localhost:5000* 网址。

在 Python 中生成 HTML 并不好玩，实际上是相当繁琐的，因为你必须自行做好 HTML 转义以保持应用程序的安全。由于这个原因，Flask 自动为你配置好 Jinja2 模版。我们将会在这一章中介绍一些模板基本概念以及基本用法。

我们接下来讲述的正是我们上一章离开的地方，所以你可能要确保应用程序 *microblog* 正确地安装和工作。


为什么我们需要模板
--------------------

让我们来考虑下我们该如何扩充我们这个小的应用程序。

我们希望我们的微博应用程序的主页上有一个欢迎登录用户的标题，这是这种类型的应用程序的一个“标配”。忽略本应用程序暂未有用户的事实，我会在后面的章节引入用户的概念。

输出一个漂亮的大标题的一个容易的选择就是改变我们的视图功能，输出 HTML，也许像这个样子::

	from app import app

	@app.route('/')
	@app.route('/index')
	def index():
	    user = { 'nickname': 'Miguel' } # fake user
	    return '''
	<html>
	  <head>
	    <title>Home Page</title>
	  </head>
	  <body>
	    <h1>Hello, ''' + user['nickname'] + '''</h1>
	  </body>
	</html>
	'''

运行看看网页浏览器上的显示情况。

我们暂时还不支持用户，所以暂时使用占位符的用户对象，有时也被称为假冒或模仿的对象。这样让我们可以集中关注应用程序的某一方面，而不用花心思在暂未完成的部分上。

我希望你同意我的说法，上面的解决方案是非常难看！如果我们需要返回一个含有大量动态内容的大型以及复杂的 HTML 页面的话，代码将会有多么复杂啊！如果你需要改变你的网站布局，在一个大的应用程序，该应用程序有几十个视图，每一个直接返回HTML？这显然​​不是一个可扩展的选择。


模板从天而降
-------------

如果你能够保持你的应用程序与网页的布局或者界面逻辑上是分开的，这样不是显得更加容易组织？难道你不觉得是这样吗？你甚至可以聘请一个网页设计师来设计一个杀手级的网页而你专注于 Python 编码。模板可以帮助实现这种分离。

让我们编写第一个我们的模板(文件 *app/templates/index.html*)::
	
	<html>
	  <head>
	    <title>{{title}} - microblog</title>
	  </head>
	  <body>
	      <h1>Hello, {{user.nickname}}!</h1>
	  </body>
	</html>

正如你在上面看到，我们只是写了一个大部分标准的HTML页面，唯一的区别是有一些动态内容的在 *{{ ... }}* 中。

现在看看怎样在我们的视图函数(文件 *app/views.py*)中使用这些模板::

	from flask import render_template
	from app import app

	@app.route('/')
	@app.route('/index')
	def index():
	    user = { 'nickname': 'Miguel' } # fake user
	    return render_template("index.html",
	        title = 'Home',
	        user = user)	

试着运行下应用程序看看模板是如何工作的。一旦在你的网页浏览器上呈现该网页，你可以浏览下 HTML 源代码，与原始的模板内容对比下差别。

为了渲染模板，我们必须从 Flask 框架中导入一个名为 *render_template* 的新函数。此函数需要传入模板名以及一些模板变量列表，返回一个所有变量被替换的渲染的模板。

在内部，*render_template* 调用了 `Jinja2 <http://jinja.pocoo.org/>`_ 模板引擎，Jinja2 模板引擎是 Flask 框架的一部分。Jinja2 会把模板参数提供的相应的值替换了 *{{...}}* 块。


模板中控制语句 
-----------------

Jinja2 模板同样支持控制语句，像在 *{%...%}* 块中。让我们在我们的模板中添加一个 if 声明(文件 *app/templates/index.html*)::

	<html>
	  <head>
	    {% if title %}
	    <title>{{title}} - microblog</title>
	    {% else %}
	    <title>Welcome to microblog</title>
	    {% endif %}
	  </head>
	  <body>
	      <h1>Hello, {{user.nickname}}!</h1>
	  </body>
	</html>

现在我们的模板变得更加智能了。如果视图函数忘记输入页面标题的参数，不会触发异常反而会出现我们自己提供的标题。放心地去掉视图函数中 *render_template* 的调用中的 *title* 参数，看看 *if* 语句是如何工作的！


模板中的循环语句
------------------

在我们 *microblog* 应用程序中，登录的用户想要在首页展示他的或者她的联系人列表中用户最近的文章，因此让我们看看如何才能做到。

首先我们先创建一些用户以及他们的文章用来展示(文件 *app/views.py*)::

	def index():
	    user = { 'nickname': 'Miguel' } # fake user
	    posts = [ # fake array of posts
	        { 
	            'author': { 'nickname': 'John' }, 
	            'body': 'Beautiful day in Portland!' 
	        },
	        { 
	            'author': { 'nickname': 'Susan' }, 
	            'body': 'The Avengers movie was so cool!' 
	        }
	    ]
	    return render_template("index.html",
	        title = 'Home',
	        user = user,
	        posts = posts)

为了表示用户的文章，我们使用了列表，其中每一个元素包含 *author* 和 *body* 字段。当我们使用真正的数据库的时候，我们会保留这些字段的名称，因此我们在设计以及测试模板的时候尽管使用的是假冒的对象，但不必担心迁移到数据库上更新模板。

在模板这一方面，我们必须解决一个新问题。列表中可能有许多元素，多少篇文章被展示将取决于视图函数。模板不会假设有多少文章，因此它必须准备渲染视图传送的文章数量。

因此让我们来看看怎么使用 *for* 来做到这一点(文件 *app/templates/index.html*)::

	<html>
	  <head>
	    {% if title %}
	    <title>{{title}} - microblog</title>
	    {% else %}
	    <title>microblog</title>
	    {% endif %}
	  </head>
	  <body>
	    <h1>Hi, {{user.nickname}}!</h1>
	    {% for post in posts %}
	    <p>{{post.author.nickname}} says: <b>{{post.body}}</b></p>
	    {% endfor %}
	  </body>
	</html>

简单吧？试试吧，确保给予足够的文章列表。


模板继承
---------

在这一章结束前我们将讨论最后一个话题。

在我们的 *microblog* 应用程序中，在页面的顶部需要一个导航栏。在导航栏里面有编辑账号，登出等等的链接。

我们可以在 *index.html* 模板中添加一个导航栏，但是随着应用的扩展，越来越多的模板需要这个导航栏，我们需要在每一个模板中复制这个导航栏。然而你必须要保证每一个导航栏都要同步，如果你有大量的模板，这需要花费很大的力气。

相反，我们可以利用 Jinja2 的模板继承的特点，这允许我们把所有模板公共的部分移除出页面的布局，接着把它们放在一个基础模板中，所有使用它的模板可以导入该基础模板。

所以让我们定义一个基础模板，该模板包含导航栏以及上面谈论的标题(文件 *app/templates/base.html*)::

	<html>
	  <head>
	    {% if title %}
	    <title>{{title}} - microblog</title>
	    {% else %}
	    <title>microblog</title>
	    {% endif %}
	  </head>
	  <body>
	    <div>Microblog: <a href="/index">Home</a></div>
	    <hr>
	    {% block content %}{% endblock %}
	  </body>
	</html>

在这个模板中，我们使用 *block* 控制语句来定义派生模板可以插入的地方。块被赋予唯一的名字。

接着现在剩下的就是修改我们的 *index.html* 模板继承自 *base.html* (文件 *app/templates/index.html*)::

	{% extends "base.html" %}
	{% block content %}
	<h1>Hi, {{user.nickname}}!</h1>
	{% for post in posts %}
	<div><p>{{post.author.nickname}} says: <b>{{post.body}}</b></p></div>
	{% endfor %}
	{% endblock %}


结束语
---------

如果你想要节省时间的话，你可以下载 `microblog-0.2.zip <https://github.com/miguelgrinberg/microblog/archive/v0.2.zip>`_。

但是请注意的是 zip 文件已经不包含 flask 虚拟环境了，如果你想要运行应用程序的话，请按照前一章的步骤自己创建它。

在下一章中，我们将会讨论到表单。我希望能在下一章继续见到各位！