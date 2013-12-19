.. _debugging:


调试，测试以及优化
====================

我们小型的 *microblog* 应用程序已经足够的完善了，因此是时候准备尽可能地清理不用的东西。近来，一个读者反映了一个奇怪的数据库问题，我们今天将会调试它。这也提醒我们不论我们是多小心以及测试我们应用程序多仔细，我们还是会遗漏一些 bug。用户是很擅长发现它们的！

不是仅仅修复此错误，然后忘记它，直到我们遇到另一个。我们会采取一些积极的措施，以更好地准备下一个。

在本章的第一部分，我们将会涉及到 *调试*，我将会展示一些我平时调试复杂问题的技巧。

接着我们想要看看如何衡量我们的测试策略的效果。我们想要清楚地知道我们的测试能够覆盖到应用程序的多少，有时候称之为 *测试覆盖率*。

最后，我们将会研究下许多应用程序遭受过的一类问题，性能糟糕。我们将会看到 *调优* 技术，并且发现我们应用程序中哪些部分最慢。

听起来不错吧？让我们开始吧！


Bug
-------

这个问题是由本系列的一个读者发现，他实现了一个允许用户删除自己的 blog 的新函数后遇到这个问题。我们正式的 *microblog* 版本是没有这个功能的，因此我们快速的加上它，以便我们可以调试这个问题。

我们使用删除 blog 的视图函数如下(文件 *app/views.py*)::

    @app.route('/delete/<int:id>')
    @login_required
    def delete(id):
        post = Post.query.get(id)
        if post == None:
            flash('Post not found.')
            return redirect(url_for('index'))
        if post.author.id != g.user.id:
            flash('You cannot delete this post.')
            return redirect(url_for('index'))
        db.session.delete(post)
        db.session.commit()
        flash('Your post has been deleted.')
        return redirect(url_for('index'))

为了调用这个函数，我们将会在模板中添加这个删除链接(文件 *app/templates/post.html*)::

    {% if post.author.id == g.user.id %}
    <div><a href="{{ url_for('delete', id = post.id) }}">{{ _('Delete') }}</a></div>
    {% endif %}

现在我们接着继续，在生产模式下运行我们的应用程序。Linux 和 Mac 用户可以这么做::

    $ ./runp.py

Windows 用户这么做::

    flask/Scripts/python runp.py

现在作为一个用户，撰写一篇 blog，接着改变主意删除它。当你点击删除链接的时候，错误出现了！

我们得到了一个简短的消息说，应用程序已经遇到了一个错误，并已通知管理员。其实这个消息就是我们的 *500.html* 模版。在生产模式下，当在处理请求的时候出现一个异常错误，Flask 会返回一个 500 错误模版给客户端。因为我们是处于生产模式下，我们看不到错误信息或者堆栈轨迹。


现场调试问题
--------------

回想 :ref:`testing` 这一章，我们开启了一些调试服务在应用程序中的生产模式版本。当时我们创造了一个写入到日志文件的日志记录器，以便应用程序可以在运行时写入调试或诊断信息。Flask 自身会在结束一个错误 500 代码请求之前写入不能处理的任何异常的堆栈轨迹。作为一个额外的选项，我们也开启了基于日志的邮件通知服务，它将会向管理员列表发送日志信息邮件当一个错误写入日志信息的时候。

因此像上面的 bug，我们能够从两个地方，日志文件和发送的邮件，获得捕获的调试信息。

一个堆栈轨迹并不足够，但是总比没有好吧。假设我们对问题一点都不知道，我们需要单从堆栈轨迹中之处发生些什么。这是这个特别的堆栈轨迹的副本::

    127.0.0.1 - - [03/Mar/2013 23:57:39] "GET /delete/12 HTTP/1.1" 500 -
    Traceback (most recent call last):
      File "/home/microblog/flask/lib/python2.7/site-packages/flask/app.py", line 1701, in __call__
        return self.wsgi_app(environ, start_response)
      File "/home/microblog/flask/lib/python2.7/site-packages/flask/app.py", line 1689, in wsgi_app
        response = self.make_response(self.handle_exception(e))
      File "/home/microblog/flask/lib/python2.7/site-packages/flask/app.py", line 1687, in wsgi_app
        response = self.full_dispatch_request()
      File "/home/microblog/flask/lib/python2.7/site-packages/flask/app.py", line 1360, in full_dispatch_request
        rv = self.handle_user_exception(e)
      File "/home/microblog/flask/lib/python2.7/site-packages/flask/app.py", line 1358, in full_dispatch_request
        rv = self.dispatch_request()
      File "/home/microblog/flask/lib/python2.7/site-packages/flask/app.py", line 1344, in dispatch_request
        return self.view_functions[rule.endpoint](**req.view_args)
      File "/home/microblog/flask/lib/python2.7/site-packages/flask_login.py", line 496, in decorated_view
        return fn(*args, **kwargs)
      File "/home/microblog/app/views.py", line 195, in delete
        db.session.delete(post)
      File "/home/microblog/flask/lib/python2.7/site-packages/sqlalchemy/orm/scoping.py", line 114, in do
        return getattr(self.registry(), name)(*args, **kwargs)
      File "/home/microblog/flask/lib/python2.7/site-packages/sqlalchemy/orm/session.py", line 1400, in delete
        self._attach(state)
      File "/home/microblog/flask/lib/python2.7/site-packages/sqlalchemy/orm/session.py", line 1656, in _attach
        state.session_id, self.hash_key))
    InvalidRequestError: Object '<Post at 0xff35e7ac>' is already attached to session '1' (this is '3')

如果你习惯读别的语言的堆栈轨迹，请注意 Python 的堆栈轨迹是反序的，最下面的是引起异常的发生点。

从上面的堆栈轨迹中我们可以看到异常是被 SQLAlchemy 会话处理代码触发的。在堆栈轨迹中，对找出我们代码最后的执行语句也是十分有帮助的。我们会最后定位到我们的 *app/views.py* 文件中的 *delete()* 函数中的 *db.session.delete(post)* 语句。

我们从上面的情况中可以知道 SQLAlchemy 是不能在数据库会话中注册删除操作。但是我们不知道为什么。

如果你仔细看看堆栈轨迹的最下面，你会发现问题好像在 *Post* 对象最开始绑定在会话 *'1'*，现在我们试着绑定同一对象到会话 *'3'* 中。

如果你搜索这个问题的话，你会发现很多用户都会遇到这个问题，尤其是使用一个多线程网页服务器，它们得到两个请求尝试同时把同一对象添加到不同的会话。但是我们使用的是 Python 开发网页服务器，它是一个单线程服务器，因此这不是我们问题的原因。这可能的原因是不知道什么操作导致两个会话在同一时间活跃。

如果我们想要了解更多的问题，我们应该尝试以一种更可控的环境下重现错误。幸运的是，在开发模式下的应用程序试图重现这个问题，它确实重现了。在开发模式中，当发生异常时，我们得到是 Flask web 的堆栈轨迹，而不是 *500.html* 模板。

web 的堆栈轨迹是十分好的，因为它允许你检查代码并且从服务器上计算表达式。没有真正理解在这段代码中到底为什么会发生这种事情，我们只是知道某些原因导致一个请求在结束的时候没有正常删除会话。因此更好的方案就是找出到底是谁创建这个会话。


使用 Python 调试器
--------------------

最容易找出谁创建一个对象的方式就是在对象构建中设置断点。断点是一个当满足一定条件的时候中断程序的命令。此时此刻，这是能够检查程序，比如获取中断时的堆栈轨迹，检查或者甚至改变变量值，等等。断点是 *调试器* 的特色之一。这次我们将会使用 Python 内置模块，叫做 *pdb*。

但是我们该检查哪个类？让我们回到基于 Web 的堆栈轨迹，再仔细找找。在最底层的堆栈帧中，我们能使用代码浏览器和 Python 控制台来找出使用会话的类。在代码中，我们看到我们是在 *Session* 类中。这像是 SQLAlchemy 中的数据库会话的基础类。因为现在在最底层的堆栈帧正是在会话对象里，我们能够在控制台中得到会话实际的类，通过运行::

  >>> print self
  <flask_sqlalchemy._SignallingSession object at 0xff34914c>

现在我们知道使用中的会话是通过 Flask-SQLAlchemy 定义的，因此这个扩展可能定义自己的会话类，作为 SQLAlchemy 会话的一个子类。

现在我们可以到 Flask-SQLAlchemy 扩展的 *flask/lib/python2.7/site-packages/flask_sqlalchemy.py* 中检查源代码并且定位到类 *_SignallingSession* 和它的 *__init__()* 构造函数，现在我们准备用调试器工作。

有很多方式在 Python 应用程序中设置断点。最简单的一种就是在我们想要中断的程序中写入如下代码::

  import pdb; pdb.set_trace()

因此我们继续向前并且暂时在 *_SignallingSession* 类的构造函数插入断点(文件 *flask/lib/python2.7/site-packages/flask_sqlalchemy.py*)::

  class _SignallingSession(Session):

      def __init__(self, db, autocommit=False, autoflush=False, **options):
          import pdb; pdb.set_trace() # <-- this is temporary!
          self.app = db.get_app()
          self._model_changes = {}
          Session.__init__(self, autocommit=autocommit, autoflush=autoflush,
                           extension=db.session_extensions,
                           bind=db.engine,
                           binds=db.get_binds(self.app), **options)

      # ...

让我们继续运行看看会发生什么::

  $ ./run.py
  > /home/microblog/flask/lib/python2.7/site-packages/flask_sqlalchemy.py(198)__init__()
  -> self.app = db.get_app()
  (Pdb)

因为没有打印出 “Running on ...” 的信息我们知道服务器实际上并没有完成启动过程。中断可能已经发生了在内部某些神秘的代码里面。

最重要的问题是我们需要回答应用程序现在处于哪里，因为这将会告诉我们谁在请求创建会话 *'1'*。我们将会使用 *bt* 来获取堆栈轨迹::

  (Pdb) bt
    /home/microblog/run.py(2)<module>()
  -> from app import app
    /home/microblog/app/__init__.py(44)<module>()
  -> from app import views, models
    /home/microblog/app/views.py(6)<module>()
  -> from forms import LoginForm, EditForm, PostForm, SearchForm
    /home/microblog/app/forms.py(4)<module>()
  -> from app.models import User
    /home/microblog/app/models.py(92)<module>()
  -> whooshalchemy.whoosh_index(app, Post)
    /home/microblog/flask/lib/python2.6/site-packages/flask_whooshalchemy.py(168)whoosh_index()
  -> _create_index(app, model))
    /home/microblog/flask/lib/python2.6/site-packages/flask_whooshalchemy.py(199)_create_index()
  -> model.query = _QueryProxy(model.query, primary_key,
    /home/microblog/flask/lib/python2.6/site-packages/flask_sqlalchemy.py(397)__get__()
  -> return type.query_class(mapper, session=self.sa.session())
    /home/microblog/flask/lib/python2.6/site-packages/sqlalchemy/orm/scoping.py(54)__call__()
  -> return self.registry()
    /home/microblog/flask/lib/python2.6/site-packages/sqlalchemy/util/_collections.py(852)__call__()
  -> return self.registry.setdefault(key, self.createfunc())
  > /home/microblog/flask/lib/python2.6/site-packages/flask_sqlalchemy.py(198)__init__()
  -> self.app = db.get_app()
  (Pdb)

像之前做的，我们会发现在 *models.py* 的 92 行中存在问题，那里是我们全文搜索引擎初始化的地方::

  whooshalchemy.whoosh_index(app, Post)

奇怪，在这个阶段我们并没有做触发数据库会话创建的事情，这看起来好像是初始化 Flask-WhooshAlchemy 的行为，它创建了一个数据库会话。

这感觉就像这毕竟不是我们的错误，也许某种形式的交互在两个 Flask 扩展 SQLAlchemy 和 Whoosh 之间。我们能停留在这里并且寻求两个扩展的开发者的帮助。或者是我们继续调试看能不能找出问题的真正所在。我将会继续，如果大家不感兴趣的话，可以跳过下面的内容。

让我们多看这个堆栈轨迹一眼。我们调用了 *whoosh_index()*，它反过来调用了 *_create_index()*。在 *_create_index()* 中的一行代码是这样的::

  model.query = _QueryProxy(model.query, primary_key,
              searcher, model)

在这里的 *model* 的变量被设置成我们的 *Post* 类，我们在调用 *whoosh_index()* 的时候传入的 *Post* 类。考虑到这一点，这看起来像是 Flask-WhooshAlchemy 创建了一个 *Post.query* 封装，它把原始的 *Post.query* 作为参数，并且附加些其它的 Whoosh 特别的东西。接着是最让人感兴趣的一部分。根据上面的堆栈轨迹，下一个调用的函数是 *__get__()*，这是一个 Python 的 `描述符 <http://docs.python.org/2/howto/descriptor.html>`_。

*__get__()* 方法是用于实现描述符，它是一个与它们行为关联的属性而不只是一个值。每次被引用,描述符 *__get__()* 的函数被调用。函数被支持返回属性的值。在这行代码中唯一被提及的的属性就是 *query*，所以现在我们知道，这个看似简单的属性，我们已经在过去使用的生成我们的数据库查询不是一个真正的属性，而是一个描述符。

让我们继续往下看看接下来发生什么。在 *__get__()* 中的代码是这个::

  return type.query_class(mapper, session=self.sa.session())

这是一个相当暴露一段代码。比如，*User.query.get(id)* 我们间接调用 *__get__()* 方法来提供查询对象，这里我们能够看到这个查询对象会暗暗地带来一个数据库会话。

当 Flask-WhooshAlchemy 使用 *model.query* 同样会触发一个会话，这个会话被创建和与查询对象关联。但是这个查询对象与运行在我们视图函数中的查询对象不一样，Flask-WhooshAlchemy 请求并不是短暂的。Flask-WhooshAlchemy 把这个查询对象传入作为自己的查询对象，并且存入到 *model.query*。由于没有 *__set__()* 方法对应，新的对象将被存储为一个属性。对于我们的 *Post* 类，这就意味着在 Flask-WhooshAlchemy 完成初始化，我们将会有名称相同的描述符和属性。根据优先级，在这种情况下，属性胜出。

这一切最重要的方面是，这段代码设置一个持久的属性，里面有我们的会话 *'1'* 。即使应用程序处理的第一个请求将使用这个会话，然后忘掉它，会话不会消失，因为它仍然是引用由 *Post.query* 属性。这是我们的错误！

该错误是由于混淆（我认为）描述的类型而造成的。它们看起来像常规属性，所以人们往往就这样使用它们。Flask-WhooshAlchemy 开发者只是想创建一个增强的查询对象用来为 Whoosh 查询存储一些有用的状态，但是他们没有意识到引用一个模型的 *query* 属性不像看起来的一样，背后隐藏与一个启动数据库的会话的属性关联。


回归测试
-----------

既然现在已经清楚了发生问题的原因所在，我们是不是可以试着重现下问题，为修复问题做一些准备。如果不愿意这么做的话，那可能只能等到 Flask-WhooshAlchemy 的开发者们去修复，那如果修复版本要等到一年以后？我们是不是要一直等待着，或者直接取消删除这个功能。

因此为了准备修复这个问题，我们可以试着去重现这个问题，我们可以试着去创建针对这个问题的测试。为了创建这个测试，我们需要模拟两个请求，第一个请求就是查询一个 Post 对象，模拟我们请求数据为了在首先显示 blog。因为这是第一个会话，我们准备命名这个会话为 *'1'*。接着我们需要忘记这个会话创建一个新的会话，就像 Flask-SQLAlchemy 所做的。试着删除 Post 对象在第二个会话中，这时候应该会触发这个 bug::

  def test_delete_post(self):
      # create a user and a post
      u = User(nickname = 'john', email = 'john@example.com')
      p = Post(body = 'test post', author = u, timestamp = datetime.utcnow())
      db.session.add(u)
      db.session.add(p)
      db.session.commit()
      # query the post and destroy the session
      p = Post.query.get(1)
      db.session.remove()
      # delete the post using a new session
      db.session = db.create_scoped_session()
      db.session.delete(p)
      db.session.commit()

现在当我们运行测试的时候失败会出现::

  $ ./tests.py
  .E....
  ======================================================================
  ERROR: test_delete_post (__main__.TestCase)
  ----------------------------------------------------------------------
  Traceback (most recent call last):
    File "./tests.py", line 133, in test_delete_post
      db.session.delete(p)
    File "/home/microblog/flask/lib/python2.7/site-packages/sqlalchemy/orm/scoping.py", line 114, in do
      return getattr(self.registry(), name)(*args, **kwargs)
    File "/home/microblog/flask/lib/python2.7/site-packages/sqlalchemy/orm/session.py", line 1400, in delete
      self._attach(state)
    File "/home/microblog/flask/lib/python2.7/site-packages/sqlalchemy/orm/session.py", line 1656, in _attach
      state.session_id, self.hash_key))
  InvalidRequestError: Object '<Post at 0xff09b7ac>' is already attached to session '1' (this is '3')

  ----------------------------------------------------------------------
  Ran 6 tests in 3.852s

  FAILED (errors=1)


修复
-------

为了解决这个问题，我们需要找到一种连接 Flask-WhooshAlchemy 查询对象到模型的替代方式。

Flask-SQLAlchemy 的文档上提到过有一个 `model.query_class <http://pythonhosted.org/Flask-SQLAlchemy/api.html#flask.ext.sqlalchemy.Model.query_class>`_ 属性包含了用于查询的类。这实际上是一个更干净的方式使得 Flask-SQLAlchemy 使用自定义的查询类而不是 Flask-WhooshAlchemy 所做的。如果我们配置 Flask-SQLAlchemy 来创建查询使用 Whoosh 启用查询类(它已经是 Flask-SQLAlchemy *BaseQuery* 的子类)，接着我们应该得到跟以前一样的结果，但是没有 bug。

我们在 github 上创建了一个 Flask-WhooshAlchemy  项目的分支，那里我已经实现上面这些修改。如果你想要看这些改变的话，请访问 `github diff <https://github.com/miguelgrinberg/Flask-WhooshAlchemy/commit/1e17350ea600e247c0094cfa4ae7145f08f4c4a3>`_，或者下载 `修改的扩展 <https://raw.github.com/miguelgrinberg/Flask-WhooshAlchemy/1e17350ea600e247c0094cfa4ae7145f08f4c4a3/flask_whooshalchemy.py>`_ 并且安装它在原始的 *flask_whooshalchemy.py* 文件所在地。


测试覆盖率
------------

虽然我们已经有了测试应用程序的测试代码，但是我们并不知道我们的应用程序有多少地方被测试到。我们需要一个测试覆盖率的工具来检查一个应用程序，在执行这个工具后我们能得到一个我们的代码现在哪些地方被测试到的报告。

Python 有一个测试覆盖率的工具，我们称之为 `coverage <http://nedbatchelder.com/code/coverage/>`_。让我们安装它::

  flask/bin/pip install coverage

这个工具可以作为一个命令行使用或者可以放在脚本里面。我们现在可以先不用考虑如何启动它。

这有些改变我们需要加入到测试代码中为了生成一个覆盖率的报告(文件 *tests.py*)::

  from coverage import coverage
  cov = coverage(branch = True, omit = ['flask/*', 'tests.py'])
  cov.start()

  # ...

  if __name__ == '__main__':
      try:
          unittest.main()
      except:
          pass
      cov.stop()
      cov.save()
      print "\n\nCoverage Report:\n"
      cov.report()
      print "HTML version: " + os.path.join(basedir, "tmp/coverage/index.html")
      cov.html_report(directory = 'tmp/coverage')
      cov.erase()

我们开始在脚本的最开始初始化 *coverage* 模块。*branch = True* 参数要求除了常规的覆盖率分析还需要做分支分析。*omit* 参数确保不会去获得我们安装在虚拟环境和测试框架自身的覆盖率报告，我们只做我们的应用程序代码的覆盖。

为了收集覆盖率统计我们只要调用 *cov.start()*，接着运行我们的单元测试。我们必须从我们的单元测试框架中捕获以及通过异常，如果脚本不结束的话是没有机会生成一个覆盖率报告。在我们从测试中回来后，我们将会用 *cov.stop()* 停止覆盖率统计，并且用 *cov.save()* 生成结果。最后，*cov.report()* 把结果输出到控制台，*cov.html_report()* 生成一个好看的 HTML 报告，*cov.erase()* 删除数据文件。

这是运行后的报告例子::

  $ ./tests.py
  .....F
      ======================================================================
  FAIL: test_translation (__main__.TestCase)
  ----------------------------------------------------------------------
  Traceback (most recent call last):
    File "./tests.py", line 143, in test_translation
      assert microsoft_translate(u'English', 'en', 'es') == u'Inglés'
  AssertionError

  ----------------------------------------------------------------------
  Ran 6 tests in 3.981s

  FAILED (failures=1)

  Coverage Report:

  Name             Stmts   Miss Branch BrMiss  Cover   Missing
  ------------------------------------------------------------
  app/__init__        39      0      6      3    93%
  app/decorators       6      2      0      0    67%   5-6
  app/emails          14      6      0      0    57%   9, 12-15, 21
  app/forms           30     14      8      8    42%   15-16, 19-30
  app/models          63      8     10      1    88%   32, 37, 47, 50, 53, 56, 78, 90
  app/momentjs        12      5      0      0    58%   5, 8, 11, 14, 17
  app/translate       33     24      4      3    27%   10-36, 39-56
  app/views          169    124     46     46    21%   16, 20, 24-30, 34-37, 41, 45-46, 53-67, 75-81, 88-109, 113-114, 120-125, 132-143, 149-164, 169-183, 188-198, 203-205, 210-211, 218
  config              22      0      0      0   100%
  ------------------------------------------------------------
  TOTAL              388    183     74     61    47%

  HTML version: /home/microblog/tmp/coverage/index.html

从上面的报告上可以看到我们测试 47% 的应用程序。我们也从上面得到没有被测试执行的函数的列表，因此我们必须重新看看这些行，考虑下我们还能编写些哪些测试。

我们能看到 *app/models.py* 覆盖率是比较高(88%),因为我们的测试集中在我们的模型。*app/views.py* 覆盖率是比较低(21%)因为我们没有在测试代码中执行视图函数。

我们新增加些测试为了提高覆盖率::

  def test_user(self):
      # make valid nicknames
      n = User.make_valid_nickname('John_123')
      assert n == 'John_123'
      n = User.make_valid_nickname('John_[123]\n')
      assert n == 'John_123'
      # create a user
      u = User(nickname = 'john', email = 'john@example.com')
      db.session.add(u)
      db.session.commit()
      assert u.is_authenticated() == True
      assert u.is_active() == True
      assert u.is_anonymous() == False
      assert u.id == int(u.get_id())

  def test_make_unique_nickname(self):
      # create a user and write it to the database
      u = User(nickname = 'john', email = 'john@example.com')
      db.session.add(u)
      db.session.commit()
      nickname = User.make_unique_nickname('susan')
      assert nickname == 'susan'
      nickname = User.make_unique_nickname('john')
      assert nickname != 'john'
      #...


性能调优
----------

下一个话题就是性能。有什么比用户等待很长时间加载页面更令人沮丧的。我们想要确保我们的应用程序的速度，我们需要一些标准或者尺寸来衡量和分析。

我们使用的技术称为 *profiling*。一个代码分析器监视正在运行的程序，很像覆盖工具，而是注意到不是行执行而是多少时间花在每个函数上。在分析阶段结束的时候会生成一个报告，里面列出了所有执行的函数以及每个函数执行了多久。对这个列表从最大到最小的时间排序是一个很好的注意，这样可以得出我们需要优化的地方。

Python 有一个称为 `cProfile <http://docs.python.org/2/library/profile.html>`_ 的代码分析器。我们能够把这个分析器直接嵌入到我们的代码中，但我们做任何工作之前，搜索是否有人已经完成了集成工作是一个好注意。一个对 “Flask profiler” 的快速搜索得出 Flask 使用的 Werkzeug 模块有一个分析器的插件，我们将直接使用它。

为了启用 Werkzeug 分析器，我们能创建一个像 *run.py* 的另外一个启动脚本。让我们称它为 *profile.py*::

  #!flask/bin/python
  from werkzeug.contrib.profiler import ProfilerMiddleware
  from app import app

  app.config['PROFILE'] = True
  app.wsgi_app = ProfilerMiddleware(app.wsgi_app, restrictions = [30])
  app.run(debug = True)

一旦这个脚本运行，每一个请求将会显示分析器的摘要。这里就是其中一个例子::

  --------------------------------------------------------------------------------
  PATH: '/'
           95477 function calls (89364 primitive calls) in 0.202 seconds

     Ordered by: internal time, call count
     List reduced from 1587 to 30 due to restriction <30>

     ncalls  tottime  percall  cumtime  percall filename:lineno(function)
          1    0.061    0.061    0.061    0.061 {method 'commit' of 'sqlite3.Connection' objects}
          1    0.013    0.013    0.018    0.018 flask/lib/python2.7/site-packages/sqlalchemy/dialects/sqlite/pysqlite.py:278(dbapi)
      16807    0.006    0.000    0.006    0.000 {isinstance}
       5053    0.006    0.000    0.012    0.000 flask/lib/python2.7/site-packages/jinja2/nodes.py:163(iter_child_nodes)
  8746/8733    0.005    0.000    0.005    0.000 {getattr}
        817    0.004    0.000    0.011    0.000 flask/lib/python2.7/site-packages/jinja2/lexer.py:548(tokeniter)
          1    0.004    0.004    0.004    0.004 /usr/lib/python2.7/sqlite3/dbapi2.py:24(<module>)
          4    0.004    0.001    0.015    0.004 {__import__}
          1    0.004    0.004    0.009    0.009 flask/lib/python2.7/site-packages/sqlalchemy/dialects/sqlite/__init__.py:7(<module>)
     1808/8    0.003    0.000    0.033    0.004 flask/lib/python2.7/site-packages/jinja2/visitor.py:34(visit)
       9013    0.003    0.000    0.005    0.000 flask/lib/python2.7/site-packages/jinja2/nodes.py:147(iter_fields)
       2822    0.003    0.000    0.003    0.000 {method 'match' of '_sre.SRE_Pattern' objects}
        738    0.003    0.000    0.003    0.000 {method 'split' of 'str' objects}
       1808    0.003    0.000    0.006    0.000 flask/lib/python2.7/site-packages/jinja2/visitor.py:26(get_visitor)
       2862    0.003    0.000    0.003    0.000 {method 'append' of 'list' objects}
    110/106    0.002    0.000    0.008    0.000 flask/lib/python2.7/site-packages/jinja2/parser.py:544(parse_primary)
         11    0.002    0.000    0.002    0.000 {posix.stat}
          5    0.002    0.000    0.010    0.002 flask/lib/python2.7/site-packages/sqlalchemy/engine/base.py:1549(_execute_clauseelement)
          1    0.002    0.002    0.004    0.004 flask/lib/python2.7/site-packages/sqlalchemy/dialects/sqlite/base.py:124(<module>)
    1229/36    0.002    0.000    0.008    0.000 flask/lib/python2.7/site-packages/jinja2/nodes.py:183(find_all)
      416/4    0.002    0.000    0.006    0.002 flask/lib/python2.7/site-packages/jinja2/visitor.py:58(generic_visit)
     101/10    0.002    0.000    0.003    0.000 flask/lib/python2.7/sre_compile.py:32(_compile)
         15    0.002    0.000    0.003    0.000 flask/lib/python2.7/site-packages/sqlalchemy/schema.py:1094(_make_proxy)
          8    0.002    0.000    0.002    0.000 {method 'execute' of 'sqlite3.Cursor' objects}
          1    0.002    0.002    0.002    0.002 flask/lib/python2.7/encodings/base64_codec.py:8(<module>)
          2    0.002    0.001    0.002    0.001 {method 'close' of 'sqlite3.Connection' objects}
          1    0.001    0.001    0.001    0.001 flask/lib/python2.7/site-packages/sqlalchemy/dialects/sqlite/pysqlite.py:215(<module>)
          2    0.001    0.001    0.002    0.001 flask/lib/python2.7/site-packages/wtforms/form.py:162(__call__)
        980    0.001    0.000    0.001    0.000 {id}
    936/127    0.001    0.000    0.008    0.000 flask/lib/python2.7/site-packages/jinja2/visitor.py:41(generic_visit)

  --------------------------------------------------------------------------------

  127.0.0.1 - - [09/Mar/2013 19:35:49] "GET / HTTP/1.1" 200 -

在这个报告中每一行的含义如下:

* **ncalls** : 这个函数被调用的次数。
* **tottime** : 在这个函数中所花费所有时间。
* **percall** : 是 **tottime** 除以 **ncalls** 的结果。
* **cumtime** : 花费在这个函数以及任何它调用的函数的时间。
* **percall** : **cumtime** 除以 **ncalls**。
* **filename\:lineno(function)** : 函数名以及位置。

有趣的是我们的模板也是作为函数出现的。这是因为 Jinja2 的模板是被编译成 Python 代码。现在看来暂时我们的应用程序还不存在性能的瓶颈。


数据库性能
-------------

为了结束这篇，我们最后看看数据库性能。从上一部分内容中数据库的处理是在性能分析的报告中，因此我们只需要在数据库变得越来越慢的时候能够获得提醒。

Flask-SQLAlchemy 文档提到了 `get_debug_queries <http://pythonhosted.org/Flask-SQLAlchemy/api.html#flask.ext.sqlalchemy.get_debug_queries>`_ 函数，它返回在请求执行期间所有的查询的列表。

这是一个很有用的信息。我们可以充分利用这个信息来得到提醒。为了充分利用这个功能，我们在配置文件中需要启动它(文件 *config.py*)::

  SQLALCHEMY_RECORD_QUERIES = True

我们需要设置一个阀值，超过这个值我们认为是一个慢的查询(文件 *config.py*)::

  # slow database query threshold (in seconds)
  DATABASE_QUERY_TIMEOUT = 0.5

为了检查是否需要发送警告，我们需要在每一个请求结束的时候进行处理。在 Flask 中，我们只需要设置一个 *after_request* 函数(文件 *app/views.py*)::

  from flask.ext.sqlalchemy import get_debug_queries
  from config import DATABASE_QUERY_TIMEOUT

  @app.after_request
  def after_request(response):
      for query in get_debug_queries():
          if query.duration >= DATABASE_QUERY_TIMEOUT:
              app.logger.warning("SLOW QUERY: %s\nParameters: %s\nDuration: %fs\nContext: %s\n" % (query.statement, query.parameters, query.duration, query.context))
      return response


结束语
---------

本章到这里就算结束了，本来打算为这个系列再写关于部署的内容，但是由于官方的教程已经很详细了，这里不再啰嗦了。有需要请访问 `部署选择 <http://www.pythondoc.com/flask/deploying/index.html>`_。

如果你想要节省时间的话，你可以下载 `microblog-0.16.zip <https://github.com/miguelgrinberg/microblog/archive/v0.16.zip>`_。

本系列准备到这里就结束了，希望大家喜欢！