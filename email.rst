.. _email:


邮件支持
===========


回顾
--------

在近来的几篇教程中，我们一直在与数据库打交道。

今天我们打算让数据库休息下，相反我们今天准备完成网页应用程序中一项重要的功能:能够给用户发送邮件。

在我们小型 *microblog* 应用程序，我们将要实现一个与邮件有关的功能，我们将会给用户发送一封邮件当他或者她被人关注的时候。实现邮件有很多方式，因此我们需要设计一个通用的框架，以便重用。


安装 Flask-Mail
-----------------

幸运地，Flask 已经存在处理邮件的扩展，尽管不是 100% 支持我们想要的功能，但是已经很好了。

在我们的虚拟环境上安装 Flask-Mail 是相当简单的。在 Windows 以外的其它系统上的用户可以按照如下命令::

    flask/bin/pip install flask-mail

Windows 上的用户稍微有些不同，因为 Flask-Mail 中使用的一个模块不支持该系统，用户需要按照如下命令::

    flask\Scripts\pip install --no-deps lamson chardet flask-mail


配置
-------

当我们在 :ref:`testing` 中，已经为了能够发送错误信息。配置了邮件相关的信息，实际上它也能用于发送用户相关的邮件。

作为提醒，我必须重复下配置中需要的信息:

* 用于发送邮件的邮件服务器
* 管理员的邮箱地址

下面就是 :ref:`testing` 中的配置(文件 *config.py*)::

    # email server
    MAIL_SERVER = 'your.mailserver.com'
    MAIL_PORT = 25
    MAIL_USE_TLS = False
    MAIL_USE_SSL = False
    MAIL_USERNAME = 'you'
    MAIL_PASSWORD = 'your-password'

    # administrator list
    ADMINS = ['you@example.com']

在开始发送应用程序的邮件之前，我们必须输入实际的邮件服务器以及管理员的邮箱。比如，如果你想要应用程序通过你的 gmail 账号发送邮件，你可以输入如下内容::

    # email server
    MAIL_SERVER = 'smtp.googlemail.com'
    MAIL_PORT = 465
    MAIL_USE_TLS = False
    MAIL_USE_SSL = True
    MAIL_USERNAME = 'your-gmail-username'
    MAIL_PASSWORD = 'your-gmail-password'

    # administrator list
    ADMINS = ['your-gmail-username@gmail.com']

我们也需要初始化一个 *Mail* 对象，这个对象为我们连接到 SMTP 服务器并且发送邮件(文件 *app/__init__.py*)::

    from flask.ext.mail import Mail
    mail = Mail(app)


让我们发送邮件！
----------------

为了学习 Flask-Mail 是如何工作的，我们将会在命令行中发送邮件。因此让我们在虚拟环境中启动 Python并且执行下面的代码::

    >>> from flask.ext.mail import Message
    >>> from app import app, mail
    >>> from config import ADMINS
    >>> msg = Message('test subject', sender = ADMINS[0], recipients = ADMINS)
    >>> msg.body = 'text body'
    >>> msg.html = '<b>HTML</b> body'
    >>> with app.app_context():
    ...     mail.send(msg)
    ....

上面的代码块将会向在配置 *config.py* 中的管理员列表所有成员发送邮件。发送者是管理员列表中的第一个管理员。邮件有文本和 HTML 版本，这取决你邮件客户端的设置。注意我们需要创建一个 *app_context* 来发送邮件。Flask-Mail 最近的版本需要这个。当 Flask 处理请求的时候，应用内容就被自动地创建。因为我们不是在请求中，必须手动地创建内容。

好了，是时候把代码整合进我们的应用程序了。


简单的邮件框架
---------------

我们现在编写一个辅助方法用于发送邮件。这是上面使用的测试代码通用的版本。我们将会把这个函数放入一个新文件，专门用于我们的应用程序的邮件支持(文件 *app/emails.py*)::

    from flask.ext.mail import Message
    from app import mail

    def send_email(subject, sender, recipients, text_body, html_body):
        msg = Message(subject, sender = sender, recipients = recipients)
        msg.body = text_body
        msg.html = html_body
        mail.send(msg)

Flask-Mail 支持的应用超出了我们所用的。比如，抄送以及附件也是可用的，但是我们不准备在应用程序中使用。


关注提醒
-----------

现在我们已经有了发送邮件的基本框架，我们可以编写发送关注提醒的函数(文件 *app/emails.py*)::

    from flask import render_template
    from config import ADMINS

    def follower_notification(followed, follower):
        send_email("[microblog] %s is now following you!" % follower.nickname,
            ADMINS[0],
            [followed.email],
            render_template("follower_email.txt", 
                user = followed, follower = follower),
            render_template("follower_email.html", 
                user = followed, follower = follower))

也许你会感到很惊奇。我们的老朋友 *render_template* 居然出现在这里。如果你还记得，我们用此函数来渲染视图中所有的 HTML 模板。像视图中的模板一样，邮件的主体也是使用模板的理想候选者。

因此我们需要编写我们的关注者提醒邮件的文本以及 HTML 版本的模板。这里是文本版本(文件 *app/templates/follower_email.txt*)::

    Dear {{user.nickname}},

    {{follower.nickname}} is now a follower. Click on the following link to visit {{follower.nickname}}'s profile page:

    {{url_for("user", nickname = follower.nickname, _external = True)}}

    Regards,

    The microblog admin

对于 HTML 版本，我们可能会做得更好些，甚至会显示出关注者的头像和用户信息(文件 *app/templates/follower_email.html*)::

    <p>Dear {{user.nickname}},</p>
    <p><a href="{{url_for("user", nickname = follower.nickname, _external = True)}}">{{follower.nickname}}</a> is now a follower.</p>
    <table>
        <tr valign="top">
            <td><img src="{{follower.avatar(50)}}"></td>
            <td>
                <a href="{{url_for('user', nickname = follower.nickname, _external = True)}}">{{follower.nickname}}</a><br />
                {{follower.about_me}}
            </td>
        </tr>
    </table>
    <p>Regards,</p>
    <p>The <code>microblog</code> admin</p>

注意在上面模板中的 *url_for* 有 *_external = True* 参数。默认情况下，*url_for* 函数生成的 URLs 是与当前页面的域名相关的。例如，*url_for("index")* 返回值将会是 */index*，但是在实际视图函数中返回的是 *http://localhost:5000/index*。在邮件中是不存在域名的内容，因此我们必须要生成完全的包含域名的 URLs，*_external* 参数就是为这个目的。

最后一步就是把发送邮件整合到实际的视图函数中(文件 *app/views.py*)::

    from emails import follower_notification

    @app.route('/follow/<nickname>')
    @login_required
    def follow(nickname):
        user = User.query.filter_by(nickname = nickname).first()
        # ...
        follower_notification(user, g.user)
        return redirect(url_for('user', nickname = nickname))

现在您可以创建两个账户并且让一个用户关注另外一个用户，看看邮件提醒是如何工作的。


这就足够了吗？
--------------

邮件提醒的工作已经完成了，但是是不是已经足够了？会不会存在一些问题了？

随着你不断地使用关注的链接，你可能会发现当你点击 *follow* 链接的时候，浏览器需要等到 2 到 3 秒的时间刷新页面，尽管邮件是正常地收到了。以前这可是瞬间完成的。

这是怎么回事了？

问题就是 Flask-Mail 发送邮件是同步的。网页服务器是被阻塞了当发送邮件的时候，直到邮件已交付响应返回给浏览器。想象下，如果当我们试图发送邮件的时候，邮件服务器是很慢，或者甚至更差，临时断线，会发生些什么？这个解决方案并不完美。

这已经成为了应用程序的一个瓶颈了，发送邮件应该是一个不会干扰到网页服务器的后台程序，所以让我们看看如何解决这个问题。


在 Python 中异步调用
----------------------

我们真正想要的就是 *send_email* 函数立即返回，发送邮件的工作移到后台处理。

事实上 Python 已经支持运行异步任务，而且有不止一种方式。*threading* 以及 *multiprocessing* 模块都可以达到这个目的。

每次我们需要发送邮件的时候启动一个进程的资源远远小于启动一个新的发送邮件的整个过程，因此把 *mail.send(msg)* 调用移入线程中(文件 *app/emails.py*)::

    from threading import Thread

    def send_async_email(msg):
        mail.send(msg)

    def send_email(subject, sender, recipients, text_body, html_body):
        msg = Message(subject, sender = sender, recipients = recipients)
        msg.body = text_body
        msg.html = html_body
        thr = Thread(target = send_async_email, args = [msg])
        thr.start()

如果现在测试点击 *follow* 链接后的速度的话，浏览器会瞬间刷新页面了。

既然异步的邮件发送功能已经实现了，如果将来我们需要实现其它异步的函数，还有什么需要改进的吗？我们需要为每一个实现异步功能的函数拷贝多线程的代码吗？这并不好。

我们可以通过实现一个 `装饰器 <http://www.python.org/dev/peps/pep-0318/>`_ 来解决这个问题。有了装饰器，上面的代码可以修改为::

    from decorators import async

    @async
    def send_async_email(msg):
        mail.send(msg)

    def send_email(subject, sender, recipients, text_body, html_body):
        msg = Message(subject, sender = sender, recipients = recipients)
        msg.body = text_body
        msg.html = html_body
        send_async_email(msg)

好的多了吧，对不对？

这个神奇的代码其实很简单。我们把它放入一个新文件(文件 *app/decorators.py*)::

    from threading import Thread

    def async(f):
        def wrapper(*args, **kwargs):
            thr = Thread(target = f, args = args, kwargs = kwargs)
            thr.start()
        return wrapper

而现在我们间接为异步任务创建了一个有用的框架，我们可以说我们已经完成了！

作为一个练习，大家可以考虑考虑如何用 *multiprocessing* 模块来实现上面的功能。
        

结束语
----------

代码中更新了本文中的一些修改，如果你想要节省时间的话，你可以下载 `microblog-0.11.zip <https://github.com/miguelgrinberg/microblog/archive/v0.11.zip>`_。

我希望能在下一章继续见到各位！