.. _testing:


单元测试
==========


回顾
------

在上一章中我们集中在一步一步为我们的应用程序的添加功能。到目前为止，我们有一个数据库功能的应用程序，它能够注册用户，允许用户登录以及登出，查看以及编辑他们的用户信息。

在本章中，我们不打算添加新的特性。相反，我们将要寻找方式来保证我们编写的代码的健壮性，我们也创建了一个测试框架用来帮助我们避免将来的失败和回归测试。


发现 bug
-------------

我记得在上一章结尾的时候，我特意提出了应用程序存在 bug。让我来描述下 bug 是什么，接着看看当不按预期工作的时候(bug 出现的时候)，我们的应用程序会发生什么。

应用程序中的 bug 就是没有有效的让我们用户的昵称保持唯一性。应用程序自动选择用户的初始昵称。如果 OpenID 提供商提供一个用户的昵称的话我们会使用这个昵称。如果没有提供话，应用程序会选择邮箱的用户名部分作为昵称。如果两个用户有着同样的昵称的话，第二个用户是不能够被注册的。更加糟糕的是，在编辑用户信息的时候，我们允许用户修改昵称，但是没有去限制昵称名称冲突。

我们将会在分析 bug 发生时候应用程序的的行为后修正这些问题。


Flask 调试
-------------

让我们来看看当我们触发一个 bug 的时候会发生些什么。

先让我们创建一个新的数据库。在 Linux::

	rm app.db
	./db_create.py

或者在 Windows 上::

	del app.db
	flask/Scripts/python db_create.py

为了重现这个 bug，你需要两个 OpenID 账号，理想地是不同的提供商，从而使得它们的 cookies 不会太复杂。遵照这些步骤创建一个重复的昵称：

	* 登录你的第一个账号
	* 进入到编辑用户信息页并且把昵称改为'dup'
	* 登出
	* 登录第二个账号
	* 进入到编辑用户信息页并且把昵称改为'dup'

糟糕！我们已经得到了来自 SQLAlchem​​y 的一个异常。错误的信息写着::

	sqlalchemy.exc.IntegrityError
	IntegrityError: (IntegrityError) column nickname is not unique u'UPDATE user SET nickname=?, about_me=? WHERE user.id = ?' (u'dup', u'', 2)

紧跟着错误后面的是错误的 `堆栈跟踪 <http://en.wikipedia.org/wiki/Stack_trace>`_，这是一个相当不错的东西，在这里你可以去任何一帧并且检查源代码或者甚至在浏览器正确地上计算表达式。

错误是相当地明显的，我们试着在数据库中插入重复的昵称。数据库模型对 *nickname* 字段有着 *unique* 限制，因此这不是一个合法的操作。

除了实际的错误，我们面前还有另外一个问题。如果一个用户不幸在我们的应用程序中遇到一个错误(这个或者其它的引起的异常)他或者她将会得到错误消息和堆栈跟踪，然而他们只是用户不是开发者。尽管这其实是一个很梦幻般的功能当我们开发的时候，但是这也是我们绝对不希望我们的用户能够看到的东西。

这段时间内我们的应用程序以调试模式运行着。调试模式是在应用程序运行的时候通过在 *run* 方法中传入参数 *debug = True*。

当我们在开发的应用程序的时候这个功能很方便，但是我们必须在生产环境上确保这个功能被禁用。让我们创建另外一个调试模式禁用的启动脚本(文件 *runp.py*)::

	#!flask/bin/python
	from app import app
	app.run(debug = False)

现在重启应用程序::

	./runp.py

接着重新尝试修改第二个账号的昵称为 ‘dup’。

这个时候我们不会得到一个前面出现的错误。相反，我们会得到一个 HTTP 错误 500，这是服务器内部错误。尽管还是返回一个错误，但至少不暴露我们的应用程序的任何细节给陌生人。当调试关闭，500 错误页是由 Flask 产生的并且发生了未处理的异常。

虽然情况有些好转，我们现在有两个新的问题。第一个是外观上的：默认的 500 错误页很丑陋。第二个小问题相当重要。我们可能不会知道什么时候用户会在我们的程序中会遇到一个失败因为现在调试被禁用。幸好有两种简单的方式解决这两个问题。


定制 HTTP 错误处理器
--------------------

Flask 为应用程序提供了一种安装自己的错误页的机制。作为例子，让我们自定义 HTTP 404 以及 500 错误页，这是最常见的两个。定义其它错误的方式是一样的。

为了声明一个定制的错误处理器，需要使用装饰器 *errorhandler* (文件 *app/views.py*)::

	@app.errorhandler(404)
	def internal_error(error):
	    return render_template('404.html'), 404

	@app.errorhandler(500)
	def internal_error(error):
	    db.session.rollback()
	    return render_template('500.html'), 500

上面的不需要多做解释，代码很清楚，唯一值得感兴趣就是在错误 500 处理器中的 *rollback* 声明。这是很有必要的因为这个函数是被作为异常的结果被调用。如果异常是被一个数据库错误触发，数据库的会话会处于一个不正常的状态，因此我们必须把会话回滚到正常工作状态在渲染 500 错误页模板之前。

这是 404 错误的模板::

	<!-- extend base layout -->
	{% extends "base.html" %}

	{% block content %}
	<h1>File Not Found</h1>
	<p><a href="{{url_for('index')}}">Back</a></p>
	{% endblock %}

这是 500 错误的一个模板::

	<!-- extend base layout -->
	{% extends "base.html" %}

	{% block content %}
	<h1>An unexpected error has occurred</h1>
	<p>The administrator has been notified. Sorry for the inconvenience!</p>
	<p><a href="{{url_for('index')}}">Back</a></p>
	{% endblock %}

注意的是在上面两个模板中我们继续使用我们 *base.html* 布局，这是为了让错误页面和应用程序的外观是统一的。


通过电子邮件发送错误
--------------------

为了解决我们第二个问题，我们将会配置两种应用程序错误报告机制。第一个就是当错误发生的时候发送电子邮件。

在开始之前我们先在应用程序中配置邮件服务器以及管理员邮箱地址(文件 *config.py*)::

	# mail server settings
	MAIL_SERVER = 'localhost'
	MAIL_PORT = 25
	MAIL_USERNAME = None
	MAIL_PASSWORD = None

	# administrator list
	ADMINS = ['you@example.com']

Flask 使用 Python *logging* 模块，因此当发生异常的时候发送邮件是十分简单(文件 *app/__init__.py*)::

	from config import basedir, ADMINS, MAIL_SERVER, MAIL_PORT, MAIL_USERNAME, MAIL_PASSWORD

	if not app.debug:
	    import logging
	    from logging.handlers import SMTPHandler
	    credentials = None
	    if MAIL_USERNAME or MAIL_PASSWORD:
	        credentials = (MAIL_USERNAME, MAIL_PASSWORD)
	    mail_handler = SMTPHandler((MAIL_SERVER, MAIL_PORT), 'no-reply@' + MAIL_SERVER, ADMINS, 'microblog failure', credentials)
	    mail_handler.setLevel(logging.ERROR)
	    app.logger.addHandler(mail_handler)

在一个没有邮件服务器的开发机器上测试上述代码是相当容易的，多亏了 Python 的 SMTP 调试服务器。仅需要打开一个新的命令行窗口(Windows 用户打开命令提示符)接着运行如下内容打开一个伪造的邮箱服务器::

	python -m smtpd -n -c DebuggingServer localhost:25

当邮箱服务器运行后，应用程序发送的邮件将会被接收到并且显示在命令行窗口上。


记录到文件
-------------

通过邮件接收错误是不错的，但是有时候这并不够。有些失败并不是结束于异常而且也不是主要问题，然而我们可能想要在日志中追踪它们以便做一些调试。

出于这个原因，我们还要为应用程序保持一个日志文件。

启用日志记录类似于电子邮件发送错误(文件 *app/__init__.py*)::

	if not app.debug:
	    import logging
	    from logging.handlers import RotatingFileHandler
	    file_handler = RotatingFileHandler('tmp/microblog.log', 'a', 1 * 1024 * 1024, 10)
	    file_handler.setFormatter(logging.Formatter('%(asctime)s %(levelname)s: %(message)s [in %(pathname)s:%(lineno)d]'))
	    app.logger.setLevel(logging.INFO)
	    file_handler.setLevel(logging.INFO)
	    app.logger.addHandler(file_handler)
	    app.logger.info('microblog startup')

日志文件将会在 *tmp* 目录，名称为 *microblog.log*。我们使用了 *RotatingFileHandler* 以至于生成的日志的大小是有限制的。在这个例子中，我们的日志文件的大小限制在 1 兆，我们将保留最后 10 个日志文件作为备份。

*logging.Formatter* 类能够定制化日志信息的格式。由于这些信息记录到一个文件中，我们希望它们提供尽可能多的信息，所以我们写一个时间戳，日志记录级别和消息起源于以及日志消息和堆栈跟踪的文件和行号。

为了使得日志更有作用，我们降低了应用程序日志以及文件日志处理器的级别，这样给我们机会写入有用的信息到日志并不是必须错误发生的时候。从这以后，每次你以非调试模式启动有用程序，日志将会记录事件。

虽然我们不会在这个时候有很多记录器的需求，调试的一个处于联机状态并在使用中的网页服务器是非常困难的。消息记录到一个文件，是一个非常有用的工具，在诊断和定位问题，所以我们现在都准备好，我们需要使用此功能。


修复 bug
-------------

让我们解决 *nickname* 重复的问题。

像之前讨论的，目前存在两个地方没有处理重复。第一个就是在 *after_login* 函数。当一个用户成功地登录进系统这个函数就会被调用，这里我们需要创建一个新的 User 实例。这里就是受影响的代码块(文件 *app/views.py*)::

   if user is None:
        nickname = resp.nickname
        if nickname is None or nickname == "":
            nickname = resp.email.split('@')[0]
        nickname = User.make_unique_nickname(nickname)
        user = User(nickname = nickname, email = resp.email, role = ROLE_USER)
        db.session.add(user)
        db.session.commit()

解决问题的方式就是让 User 类为我们选择一个唯一的名字。这就是新的 *make_unique_nickname* 方法所做的(文件 *app/models.py*)::

    class User(db.Model):
    # ...
    @staticmethod
    def make_unique_nickname(nickname):
        if User.query.filter_by(nickname = nickname).first() == None:
            return nickname
        version = 2
        while True:
            new_nickname = nickname + str(version)
            if User.query.filter_by(nickname = new_nickname).first() == None:
                break
            version += 1
        return new_nickname
    # ...

这种方法简单地增加一个计数器为请求的昵称，直到找到一个唯一的名称。例如，如果用户名 “miguel”已经存在，这个方法将会建议使用 “miguel2”，如果这个还是存在，将会建议使用 "miguel3"，依次下去直至找到唯一的用户名。需要注意的是我们把这个方法作为一个静态方法，因为这种操作并不适用于任何特定的类的实例。
	
第二个存在重复昵称问题的地方就是编辑用户信息的视图函数。这个稍微有些难处理，因为这是用户自己选择的昵称。正确的做法就是不接受一个重复的昵称，让用户重新输入一个。我们将通过添加一个昵称表单字段定制化的验证来解决这个问题。如果用户输入一个不合法的昵称，字段的验证将会失败，用户将会返回到编辑用户信息页。为了添加验证，我们只需覆盖表单的 *validate* 方法(文件 *app/forms.py*)::

	from app.models import User

	class EditForm(Form):
	    nickname = TextField('nickname', validators = [Required()])
	    about_me = TextAreaField('about_me', validators = [Length(min = 0, max = 140)])

	    def __init__(self, original_nickname, *args, **kwargs):
	        Form.__init__(self, *args, **kwargs)
	        self.original_nickname = original_nickname

	    def validate(self):
	        if not Form.validate(self):
	            return False
	        if self.nickname.data == self.original_nickname:
	            return True
	        user = User.query.filter_by(nickname = self.nickname.data).first()
	        if user != None:
	            self.nickname.errors.append('This nickname is already in use. Please choose another one.')
	            return False
	        return True

表单的初始化新增了一个参数 *original_nickname*。*validate* 方法使用它来决定昵称什么时候更改过。如果没有发生更改就接受它。如果已经发生更改的话，确保昵称在数据库是唯一的。

在视图函数中传入这个参数::

	@app.route('/edit', methods = ['GET', 'POST'])
	@login_required
	def edit():
	    form = EditForm(g.user.nickname)
	    # ...

为了完成这个修改，我们必须在表单模板中使得字段错误信息会显示(文件 *app/templates/edit.html*)::

    <td>Your nickname:</td>
    <td>
        {{form.nickname(size = 24)}}
        {% for error in form.errors.nickname %}
        <br><span style="color: red;">[{{error}}]</span>
        {% endfor %}
    </td>

现在问题是修复了，重复将会被禁止。。。除非是都没有。我们仍然存在潜在的问题，当两个或者更多的线程或者处理同时访问数据库的时候，但是这将会是以后的话题。


单元测试框架
--------------

在结束本章的话题之前，让我们来讨论一点自动化测试。

随着应用程序的规模变得越大就越难保证代码的修改不会影响到现有的功能。

传统的方式--回归测试是一个很好的主意。你编写测试检验应用程序所有不同的功能。每一个测试集中在一个关注点上验证结果是不是期望的。定期执行测试确保应用程序按预期的工作。当测试覆盖很大的时候，通过运行测试你就有自信确保修改点和新增点不会影响应用程序。

我们使用 Python 的 *unittest* 模块将会构建一个简单的测试框架(文件 *tests.py*)::

	#!flask/bin/python
	import os
	import unittest

	from config import basedir
	from app import app, db
	from app.models import User

	class TestCase(unittest.TestCase):
	    def setUp(self):
	        app.config['TESTING'] = True
	        app.config['CSRF_ENABLED'] = False
	        app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' + os.path.join(basedir, 'test.db')
	        self.app = app.test_client()
	        db.create_all()

	    def tearDown(self):
	        db.session.remove()
	        db.drop_all()

	    def test_avatar(self):
	        u = User(nickname = 'john', email = 'john@example.com')
	        avatar = u.avatar(128)
	        expected = 'http://www.gravatar.com/avatar/d4c74594d841139328695756648b6bd6'
	        assert avatar[0:len(expected)] == expected

	    def test_make_unique_nickname(self):
	        u = User(nickname = 'john', email = 'john@example.com')
	        db.session.add(u)
	        db.session.commit()
	        nickname = User.make_unique_nickname('john')
	        assert nickname != 'john'
	        u = User(nickname = nickname, email = 'susan@example.com')
	        db.session.add(u)
	        db.session.commit()
	        nickname2 = User.make_unique_nickname('john')
	        assert nickname2 != 'john'
	        assert nickname2 != nickname

	if __name__ == '__main__':
	    unittest.main()

讨论 *unittest* 模块是在本文的范围之外的。*TestCase* 类中含有我们的测试。*setUp* 和 *tearDown* 方法是特别的，它们分别在测试之前以及测试之后运行。

在上面代码中 *setUp* 和 *tearDown* 方法十分普通。在 *setUp* 中做了一些配置，在 *tearDown* 中重置数据库内容。

测试实现成了方法。一个测试支持运行应用程序的多个函数，并且有已知的结果以及应该断言结果是否不同于预期的。

目前为止在测试框架中有两个测试。第一个就是验证 Gravatar 的头像 URL生成是否正确。注意测试中期待的结果是硬编码，验证 *User* 类的返回的头像 URL。

第二个就是我们前面编写的 *make_unique_nickname* 方法，同样是在 *User* 类中。


结束语
--------

如果你想要节省时间的话，你可以下载 `microblog-0.7.zip <https://github.com/miguelgrinberg/microblog/archive/v0.7.zip>`_。

我希望能在下一章继续见到各位！