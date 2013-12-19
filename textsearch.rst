.. _textsearch:


全文搜索
============


回顾
--------

在前面的章节(:ref:`pagination`)，我们已经加强了数据库查询，因此能够在页面上获取各种查询。

今天，我们会继续探讨数据库的话题，只是领域不同。所有存储内容的应用程序必须提供搜索能力。

许多其它类型的网站可能使用了谷歌、必应等索引所有的内容并且提供查询结果。这个对于大多数静态页面的网站，像论坛，是很好用。我们应用程序 *microblog* 的基本单元是用户短小的 blog，不是整个页面。我们希望搜索结果是动态的。例如，我们想要在所有的 blog 中搜索关键词 “dog”。这是显而易见的，除非是有人搜索这个词，不然大型的搜索引擎不可能索引搜索结果。因此我们除了使用自己的搜索是别无选择的。


全文搜索引擎的简介
----------------------

不幸的是，在关系数据库中的全文搜索支持没有得到很好的规范。每个数据库都以自己的方式实现全文搜索，并且 SQLAlchemy 没有实现全文搜索。

我们目前使用了 SQLite 作为数据库，因此我们可以绕过 SQLAlchemy，使用 SQLite 提供的特性来创建全文文本索引。但是这并不是一个好主意，因为如果我们要更换数据库的时候，我们需要重写全文搜索的代码。

因此，相反我们让数据库处理常规数据，我们将创建一个专门的数据库，专注服务于文本搜索。

现在有一些开源的全文搜索引擎。在我的知识范围内唯一一个用 Python 编写的 Flask 扩展是 `Whoosh <https://bitbucket.org/mchaput/whoosh/wiki/Home>`_。一个纯 Python 的搜索引擎的好处就是在 Python 解释器可用的任何地方能够安装和运行。缺点也是很显然的，性能可能比不上 C 或者 C++ 编写的。我的观点是最理想的解决方式就是开发一个连接不同搜索引擎的 Flask 扩展，以某种方式来处理搜索，就像 Flask-SQLAlchemy 一样。但是目前在 Flask 中暂时没有这类型的扩展。Django 开发者提供了一个很好的扩展，用来支持不同的全文搜索引擎，叫做 `django-haystack <http://haystacksearch.org/>`_。也许不久就会有人写一个类似的 Flask 扩展。

如果你暂时没有在虚拟环境上安装 Flask-WhooshAlchemy，请安装它。Windows 用户应该运行这个::

    flask\Scripts\pip install Flask-WhooshAlchemy

其它用户必须运行这个::

    flask/bin/pip install Flask-WhooshAlchemy


配置
---------

配置 Flask-WhooshAlchemy 也是相当简单。我们只需要告诉扩展全文搜索数据库的名称(文件 *config.py*)::

    WHOOSH_BASE = os.path.join(basedir, 'search.db')


模型修改
----------

因为把 Flask-WhooshAlchemy 整合进 Flask-SQLAlchemy，我们需要在模型的类中指明哪些数据需要建立搜索索引(文件 *app/models.py*)::

    from app import app
    import flask.ext.whooshalchemy as whooshalchemy

    class Post(db.Model):
        __searchable__ = ['body']

        id = db.Column(db.Integer, primary_key = True)
        body = db.Column(db.String(140))
        timestamp = db.Column(db.DateTime)
        user_id = db.Column(db.Integer, db.ForeignKey('user.id'))

        def __repr__(self):
            return '<Post %r>' % (self.body)

    whooshalchemy.whoosh_index(app, Post)

模型有一个新的 *__searchable__* 字段，这里面包含数据库中的所有能被搜索并且建立索引的字段。在我们的例子中，我们只要索引 blog 的 *body* 字段。

通过调用 *whoosh_index* 函数，我们为这个模型初始化了全文搜索索引。

因为这个改变并不影响到关系数据库的格式，因此不需要录制新的迁移脚本。

因为之前存储在数据库的 blog 是没有建立索引的。为了保持数据库和全文搜索引擎的同步，我们需要删除之前撰写的 blog::

    >>> from app.models import Post
    >>> from app import db
    >>> for post in Post.query.all():
    ...    db.session.delete(post)
    >>> db.session.commit()


搜索
-------

现在我们准备开始搜索。首先让我们在数据库中添加些 blog。有两种方式去添加。我们可以运行应用程序，通过浏览器像普通用户一样添加 blog。另外一种就是在 Python 提示符下。

在 Python 提示符下，我们可以按如下的去做::

    >>> from app.models import User, Post
    >>> from app import db
    >>> import datetime
    >>> u = User.query.get(1)
    >>> p = Post(body='my first post', timestamp=datetime.datetime.utcnow(), author=u)
    >>> db.session.add(p)
    >>> p = Post(body='my second post', timestamp=datetime.datetime.utcnow(), author=u)
    >>> db.session.add(p)
    >>> p = Post(body='my third and last post', timestamp=datetime.datetime.utcnow(), author=u)
    >>> db.session.add(p)
    >>> db.session.commit()

现在我们在全文索引中有一些 blog，我们可以这样搜索::

    >>> Post.query.whoosh_search('post').all()
    [<Post u'my second post'>, <Post u'my first post'>, <Post u'my third and last post'>]
    >>> Post.query.whoosh_search('second').all()
    [<Post u'my second post'>]
    >>> Post.query.whoosh_search('second OR last').all()
    [<Post u'my second post'>, <Post u'my third and last post'>]

在上面例子中你可以看到，查询并不限制于单个词。实际上，Whoosh 支持一个更加强大的 `搜索查询语言 <http://packages.python.org/Whoosh/querylang.html>`_。


整合全文搜索到应用程序
------------------------

为了使得搜索功能在我们的应用程序中可用，我们需要添加些修改。

配置
^^^^^^^

在配置文件中，我们需要指明搜索结果返回的最大数量(文件 *config.py*)::

    MAX_SEARCH_RESULTS = 50

搜索表单
^^^^^^^^^

我们准备在导航栏中添加一个搜索表单。把表单放在导航栏中是有好处的，因为应用程序所有页都有搜索表单。

首先，我们添加一个搜索表单类(文件 *app/forms.py*)::

    class SearchForm(Form):
        search = TextField('search', validators = [Required()])

接着我们必须创建一个搜索表单对象并且使得它对所有模版中可用，因为我们将搜索表单放在导航栏中，导航栏是所有页面共有的。最容易的方式就是在 *before_request* 函数中创建这个表单对象，接着把它放在全局变量 *g* 中(文件 *app/views.py*)::

    from forms import SearchForm

    @app.before_request
    def before_request():
        g.user = current_user
        if g.user.is_authenticated():
            g.user.last_seen = datetime.utcnow()
            db.session.add(g.user)
            db.session.commit()
            g.search_form = SearchForm()

我们接着添加表单到模板中(文件 *app/templates/base.html*)::

    <div>Microblog:
        <a href="{{ url_for('index') }}">Home</a>
        {% if g.user.is_authenticated() %}
        | <a href="{{ url_for('user', nickname = g.user.nickname) }}">Your Profile</a>
        | <form style="display: inline;" action="{{url_for('search')}}" method="post" name="search">{{g.search_form.hidden_tag()}}{{g.search_form.search(size=20)}}<input type="submit" value="Search"></form>
        | <a href="{{ url_for('logout') }}">Logout</a>
        {% endif %}
    </div>

注意，只有当用户登录后，我们才会显示搜索表单。*before_request* 函数仅仅当用户登录才会创建一个表单对象，因为我们的程序不会对非认证用户显示任何内容。

搜索视图函数
^^^^^^^^^^^^^^^

上面的模版中，我们在 *action* 字段中设置发送搜索请求到 *search* 视图函数。*search* 视图函数如下(文件 *app/views.py*)::

    @app.route('/search', methods = ['POST'])
    @login_required
    def search():
        if not g.search_form.validate_on_submit():
            return redirect(url_for('index'))
        return redirect(url_for('search_results', query = g.search_form.search.data))

这个函数实际做的事情不多，它只是从查询表单这能够获取查询的内容，并把它作为参数重定向另外一页。搜索工作不在这里直接做的原因还是担心用户无意中触发了刷新，这样会导致表单数据被重复提交。


搜索结果页
-------------

一旦查询的关键字被接收到，*search_results* 函数就会开始工作(文件 *app/views.py*)::

    from config import MAX_SEARCH_RESULTS

    @app.route('/search_results/<query>')
    @login_required
    def search_results(query):
        results = Post.query.whoosh_search(query, MAX_SEARCH_RESULTS).all()
        return render_template('search_results.html',
            query = query,
            results = results)

搜索结果视图函数把查询传递给 Whoosh，并且把最大的结果数也作为参数传递给 Whoosh。

最后一部分就是搜索结果的模版(文件 *app/templates/search_results.html*)::

    <!-- extend base layout -->
    {% extends "base.html" %}

    {% block content %}
    <h1>Search results for "{{query}}":</h1>
    {% for post in results %}
        {% include 'post.html' %}
    {% endfor %}
    {% endblock %}


结束语
---------

如果你想要节省时间的话，你可以下载 `microblog-0.10.zip <https://github.com/miguelgrinberg/microblog/archive/v0.10.zip>`_。

我希望能在下一章继续见到各位！