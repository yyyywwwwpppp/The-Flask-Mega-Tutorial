.. _userlogin:

用户登录
==========

回顾
--------

在上一章中，我们已经创建了数据库以及学会了使用它来存储用户以及 blog，但是我们并没有把它融入我们的应用程序中。在两章以前，我们已经看到如何创建表单并且留下了一个完全实现的登录表单。

在本章中我们将会建立 web 表单和数据库的联系，并且编写我们的登录系统。在本章结尾的时候，我们这个小型的应用程序将能够注册新用户并且能够登入和登出。

我们接下来讲述的正是我们上一章离开的地方，所以你可能要确保应用程序 *microblog* 正确地安装和工作。


配置
-------

像以前章节一样，我们从配置将会使用到的 Flask 扩展开始入手。对于登录系统，我们将会使用到两个扩展，Flask-Login 和 Flask-OpenID。配置情况如下(文件 *app\__init__.py*)::

    import os
    from flask.ext.login import LoginManager
    from flask.ext.openid import OpenID
    from config import basedir

    lm = LoginManager()
    lm.init_app(app)
    oid = OpenID(app, os.path.join(basedir, 'tmp'))

Flask-OpenID 扩展需要一个存储文件的临时文件夹的路径。对此，我们提供了一个 *tmp* 文件夹的路径。


重构用户模型
-------------------

Flask-Login 扩展需要在我们的 *User* 类中实现一些特定的方法。但是类如何去实现这些方法却没有什么要求。

下面就是我们为  Flask-Login 实现的 *User* 类(文件 *app/models.py*)::

    class User(db.Model):
        id = db.Column(db.Integer, primary_key = True)
        nickname = db.Column(db.String(64), unique = True)
        email = db.Column(db.String(120), unique = True)
        role = db.Column(db.SmallInteger, default = ROLE_USER)
        posts = db.relationship('Post', backref = 'author', lazy = 'dynamic')

        def is_authenticated(self):
            return True

        def is_active(self):
            return True

        def is_anonymous(self):
            return False

        def get_id(self):
            return unicode(self.id)

        def __repr__(self):
            return '<User %r>' % (self.nickname)

*is_authenticated* 方法有一个具有迷惑性的名称。一般而言，这个方法应该只返回 *True*，除非表示用户的对象因为某些原因不允许被认证。

*is_active* 方法应该返回 *True*，除非是用户是无效的，比如因为他们的账号是被禁止。

*is_anonymous* 方法应该返回 *True*，除非是伪造的用户不允许登录系统。

最后，*get_id* 方法应该返回一个用户唯一的标识符，以 unicode 格式。我们使用数据库生成的唯一的 id。


user_loader 回调
--------------------

现在我们已经准备好用 Flask-Login 和 Flask-OpenID 扩展来开始实现登录系统。

首先，我们必须编写一个函数用于从数据库加载用户。这个函数将会被 Flask-Login 使用(文件 *app/views.py*)::

    @lm.user_loader
    def load_user(id):
        return User.query.get(int(id))

请注意在 Flask-Login 中的用户 ids 永远是 unicode 字符串，因此在我们把 id 发送给 Flask-SQLAlchemy 之前，把 id 转成整型是必须的，否则会报错！


登录视图函数
---------------

接下来我们需要更新我们的登录视图函数(文件 *app/views.py*)::

    from flask import render_template, flash, redirect, session, url_for, request, g
    from flask.ext.login import login_user, logout_user, current_user, login_required
    from app import app, db, lm, oid
    from forms import LoginForm
    from models import User, ROLE_USER, ROLE_ADMIN

    @app.route('/login', methods = ['GET', 'POST'])
    @oid.loginhandler
    def login():
        if g.user is not None and g.user.is_authenticated():
            return redirect(url_for('index'))
        form = LoginForm()
        if form.validate_on_submit():
            session['remember_me'] = form.remember_me.data
            return oid.try_login(form.openid.data, ask_for = ['nickname', 'email'])
        return render_template('login.html', 
            title = 'Sign In',
            form = form,
            providers = app.config['OPENID_PROVIDERS'])

注意我们这里导入了不少新的模块，一些模块我们将会在不久后使用到。

跟之前的版本的改动是非常小的。我们在视图函数上添加一个新的装饰器。*oid.loginhandle* 告诉 Flask-OpenID 这是我们的登录视图函数。

在函数开始的时候，我们检查 *g.user* 是否被设置成一个认证用户，如果是的话将会被重定向到首页。这里的想法是如果是一个已经登录的用户的话，就不需要二次登录了。

Flask 中的 *g* 全局变量是一个在请求生命周期中用来存储和共享数据。我敢肯定你猜到了，我们将登录的用户存储在这里(*g*)。

我们在 *redirect* 调用中使用的 *url_for* 函数是定义在 Flask 中，以一种干净的方式为一个给定的视图函数获取 URL。如果你想要重定向到首页你可能会经常使用 *redirect('/index')*，但是有很多 `好理由 <http://flask.pocoo.org/docs/quickstart/#url-building>`_ 让 Flask 为你构建 URLs。

当我们从登录表单获取的数据后的处理代码也是新的。这里我们做了两件事。首先，我们把 *remember_me* 布尔值存储到 flask 的会话中，这里别与 Flask-SQLAlchemy 中的 *db.session* 弄混淆。之前我们已经知道 *flask.g* 对象在请求整个生命周期中存储和共享数据。*flask.session* 提供了一个更加复杂的服务对于存储和共享数据。一旦数据存储在会话对象中，在来自同一客户端的现在和任何以后的请求都是可用的。数据保持在会话中直到会话被明确地删除。为了实现这个，Flask 为我们应用程序中每一个客户端设置不同的会话文件。

在接下来的代码行中，*oid.try_login* 被调用是为了触发用户使用 Flask-OpenID 认证。该函数有两个参数，用户在 web 表单提供的 *openid* 以及我们从 OpenID 提供商得到的数据项列表。因为我们已经在用户模型类中定义了 *nickname* 和 *email*，这也是我们将要从 OpenID 提供商索取的。

OpenID 认证异步发生。如果认证成功的话，Flask-OpenID 将会调用一个注册了 *oid.after_login* 装饰器的函数。如果失败的话，用户将会回到登陆页面。


Flask-OpenID 登录回调
----------------------------

这里就是我们的 *after_login* 函数的实现(文件 *app/views.py*)::

    @oid.after_login
    def after_login(resp):
        if resp.email is None or resp.email == "":
            flash('Invalid login. Please try again.')
            return redirect(url_for('login'))
        user = User.query.filter_by(email = resp.email).first()
        if user is None:
            nickname = resp.nickname
            if nickname is None or nickname == "":
                nickname = resp.email.split('@')[0]
            user = User(nickname = nickname, email = resp.email, role = ROLE_USER)
            db.session.add(user)
            db.session.commit()
        remember_me = False
        if 'remember_me' in session:
            remember_me = session['remember_me']
            session.pop('remember_me', None)
        login_user(user, remember = remember_me)
        return redirect(request.args.get('next') or url_for('index'))

*resp* 参数传入给 after_login 函数，它包含了从 OpenID 提供商返回来的信息。

第一个 *if* 只是为了验证。我们需要一个合法的邮箱地址，因此提供邮箱地址是不能登录的。

接下来，我们从数据库中搜索邮箱地址。如果邮箱地址不在数据库中，我们认为是一个新用户，因为我们会添加一个新用户到数据库。注意例子中我们处理空的或者没有提供的 *nickname* 方式，因为一些 OpenID 提供商可能没有它的信息。

接着，我们从 flask 会话中加载 *remember_me* 值，这是一个布尔值，我们在登录视图函数中存储的。

然后，为了注册这个有效的登录，我们调用 Flask-Login 的 *login_user* 函数。

最后，如果在 next 页没有提供的情况下，我们会重定向到首页，否则会重定向到 next 页。

如果要让这些都起作用的话，Flask-Login 需要知道哪个视图允许用户登录。我们在应用程序模块初始化中配置(文件 *app/__init__.py*)::

    lm = LoginManager()
    lm.init_app(app)
    lm.login_view = 'login'


全局变量 *g.user*
---------------------

如果你观察仔细的话，你会记得在登录视图函数中我们检查 *g.user* 为了决定用户是否已经登录。为了实现这个我们用 Flask 的 *before_request* 装饰器。任何使用了 *before_request* 装饰器的函数在接收请求之前都会运行。 因此这就是我们设置我们 *g.user* 的地方(文件 *app/views.py*)::

    @app.before_request
    def before_request():
        g.user = current_user

这就是所有需要做的。全局变量 *current_user* 是被 Flask-Login 设置的，因此我们只需要把它赋给 *g.user* ，让访问起来更方便。有了这个，所有请求将会访问到登录用户，即使在模版里。


首页视图
-------------

在前面的章节中，我们的 *index* 视图函数使用了伪造的对象，因为那时候我们并没有用户或者 blog。好了，现在我们有用户了，让我们使用它::

    @app.route('/')
    @app.route('/index')
    @login_required
    def index():
        user = g.user
        posts = [
            { 
                'author': { 'nickname': 'John' }, 
                'body': 'Beautiful day in Portland!' 
            },
            { 
                'author': { 'nickname': 'Susan' }, 
                'body': 'The Avengers movie was so cool!' 
            }
        ]
        return render_template('index.html',
            title = 'Home',
            user = user,
            posts = posts)

上面仅仅只有两处变化。首先，我们添加了 *login_required* 装饰器。这确保了这页只被已经登录的用户看到。

另外一个变化就是我们把 *g.user* 传入给模版，代替之前使用的伪造对象。

这是运行应用程序最好的时候了！


登出
-------

我们已经实现了登录，现在是时候增加登出的功能。

登出的视图函数是相当地简单(文件 *app/views.py*)::

    @app.route('/logout')
    def logout():
        logout_user()
        return redirect(url_for('index'))

但是我们还没有在模版中添加登出的链接。我们将要把这个链接放在基础层中的导航栏里(文件 *app/templates/base.html*)::

    <html>
      <head>
        {% if title %}
        <title>{{title}} - microblog</title>
        {% else %}
        <title>microblog</title>
        {% endif %}
      </head>
      <body>
        <div>Microblog:
            <a href="{{ url_for('index') }}">Home</a>
            {% if g.user.is_authenticated() %}
            | <a href="{{ url_for('logout') }}">Logout</a>
            {% endif %}
        </div>
        <hr>
        {% with messages = get_flashed_messages() %}
        {% if messages %}
        <ul>
        {% for message in messages %}
            <li>{{ message }} </li>
        {% endfor %}
        </ul>
        {% endif %}
        {% endwith %}
        {% block content %}{% endblock %}
      </body>
    </html>

实现起来是不是很简单？我们只需要检查有效的用户是否被设置到 *g.user* 以及是否我们已经添加了登出链接。我们也正好利用这个机会在模版中使用 *url_for*。


结束语
-----------

我们现在已经有一个完全实现的登录系统。在下一章中，我们将会创建用户信息页以及将会显示用户头像。

如果你想要节省时间的话，你可以下载 `microblog-0.5.zip <https://github.com/miguelgrinberg/microblog/archive/v0.5.zip>`_。



