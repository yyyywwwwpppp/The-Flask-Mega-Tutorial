.. _dateandtime:


日期和时间
=============


善意提醒
-----------

对于那些还没有注意到的读者，近来项目已经迁移到 github 上，你们可以在这个位置找到代码: https://github.com/miguelgrinberg/microblog。

我已经添加了标签指向每个教程步​​骤，为您提供便利。


时间戳的问题
---------------

我们 *microblog* 应用程序已经忽略很长时间的一个方面就是时间和日期的显示。

到目前为止，我们信任 Python 本身去渲染在我们 *User* 和 *Post* 对象中的 *datetime* 对象，然而这真不是一个好的解决方案。

考虑如下的例子。我在 2012.12.31 的下午 3:54 写的这篇文章。我的时区是 PST (或者 UTC-8 如果你更喜欢的话)。在 Python 解释器运行能得到如下信息::

    >>> from datetime import datetime
    >>> now = datetime.now()
    >>> print now
    2012-12-31 15:54:42.915204
    >>> now = datetime.utcnow()
    >>> print now
    2012-12-31 23:55:13.635874

*now()* 调用返回本地时区的正确的时间，然而 *utcnow()* 调用返回 UTC 单位的时间。

因此用哪个更好？

如果我们所有的时间戳都使用 *now()*，我们将会在数据库中存储服务器运行本地的时间，然而这会带来一些问题。

有一天我们想把服务器迁移到不同时区的地方，那么所有数据库中的时间戳必须修改成当地的正确时间在服务器重启之前。

但是这还有一个更重要的问题。对于不同时区的用户很难清楚知道 blog 的发布时间因为他们看到的是在 PST 时区的时间。他们需要提前知道服务器的时区是 PST 才能做出适当的调整。

显然，这不是一个好的选择，这是为什么开始使用我们的数据库的时候，我们决定我们总是以 UTC 时区存储时间戳。

尽管用 UCT 时区标准化了时间戳解决了迁移服务器的问题。但是它解决不了第二个问题，日期和时间在世界上的任何地方都会以 UTC 形式呈献给用户。

这也是很令人困惑的。想象下一个在 PST 时区的用户在下午 3:00 发布一篇 blog，结果首页显示的时间是下午 11:00 或者是 23:00。用户会不会感到很奇怪啊？

今天，文章的目的是解决日期和时间显示的问题，使得我们的用户不要混淆。


用户特定的时间戳
-------------------

最明显解决问题的方案就是为每一个用户单独把时间戳从 UTC 转换到当地时区。这也允许我们接着在数据库中使用 UTC 时区。

但是如何知道每一个用户的当地时区了？

许多网页都会有一个配置页，用户可以指定当地时区。这就需要我们添加一个新页，新页上需要一个表单，用户可以在表单上的下拉框中选择自己当地的时区。作为注册的一部分，用户第一次访问页面的时候，需要被要求输入他们的时区。

尽管这是一个比较好的解决方案，但是用户的系统配置中已经配置了时区，让用户输入这样的信息有些累赘。如果我们能获取用户电脑上的时区的话看起来更有效些。

出于安全原因，网页浏览器是不允许从用户的操作系统中获取信息。即使这是可能的话，我们需要在 Windows, Linux, Mac, iOS, 以及 Android 上(这还是没有统计其他类型的操作系统)去查询时区。

其实网页浏览器知道用户的时区，并且可以通过标准的 Javascript APIs 导出。在 Web 2.0 世界里，用户开启 Javascript 的假设是安全的(现代的网页没有脚本是不能工作的)，因此这个方案是有潜力的。

我们有两种方式通过 Javascript 来利用可用的时区配置:

* “老派” 的方式就是让网页浏览器在某时候发送时区信息当用户第一次登录到服务器。这可以通过 `Ajax <http://en.wikipedia.org/wiki/Ajax_(programming)>`_ 调用，或者 一个 `刷新标记 <http://en.wikipedia.org/wiki/Meta_refresh>`_。一旦服务器知道了时区会把它保存到用户的会话中，当模板渲染的时候会去调整时间。
* “新派” 的方式就是不会在服务器上改变任何东西，会继续把 UTC 时间戳发送到客户端浏览器上。从 UTC 到当地时区的转换是通过 Javascript 在客户端实现的。

两种方式都不错，但是第二种更加不错。浏览器是最有能力根据系统时区配置来渲染时间的。最具有吸引力的是，第二种方式有现成的。


介绍 moment.js
-----------------

`Moment.js <http://momentjs.com/>`_ 是一个小型的，自由的，开源的 Javascript 库，它能够渲染日期和时间。它提供了富有想象的格式化选项。

为了在我们应用程序中使用 moment.js，我们需要在模板中写一点 Javascript。我们开始从 `ISO 8601 <http://en.wikipedia.org/wiki/ISO_8601>`_ 时区构建一个 *moment* 对象。例如，我们可以创建一个 moment 对象像这样:

    moment("2012-12-31T23:55:13 Z")

一旦对象被构建，它能够被渲染成各种格式的字符串。例如，根据系统时区进行详细的渲染::

    moment("2012-12-31T23:55:13 Z").format('LLLL');

下面就是结果::

    Tuesday, January 1 2013 7:55 AM

这里是一些不同格式渲染的效果:

.. image:: images/7.jpg

除了提供了 *format()*，它还提供了 *fromNow()* 和 *calendar()*:

.. image:: images/8.jpg

请注意，在所有的例子中，服务器渲染相同的 UTC 时间，在自己的网页浏览器中执行不同的渲染。

我们现在缺少的就是让 *moment* 返回的字符串在页面上可见。实现这个最简单的方式就是 Javascript 的 *document.write* 函数::

    <script>
    document.write(moment("2012-12-31T23:55:13 Z").format('LLLL'));
    </script>


整合 moment.js
------------------

在我们的应用程序中使用 *moment.js* 还需要做一些事情。

首先，把下载下来的 *moment.min.js* 放入到 */app/static/js* 文件夹，这样它就能作为一个静态文件提供给客户端。

接着，在我们的基础模板中添加对这个库的引用(文件 *app/templates/base.html*)::

    <script src="/static/js/moment.min.js"></script>

我们现在在模板中添加 *<script>* 标签，用来显示时间戳。但是与这样方式不同的，我们将会创建一个 *moment.js* 封装，我们能在模版中调用这个封装。这会节省不少时间如果我们必须修改时间戳渲染代码，因为我们已经把它放在一个地方。

我们的封装是一个很简单的 Python 类(文件 *app/momentjs.py*)::

    from jinja2 import Markup

    class momentjs(object):
        def __init__(self, timestamp):
            self.timestamp = timestamp

        def render(self, format):
            return Markup("<script>\ndocument.write(moment(\"%s\").%s);\n</script>" % (self.timestamp.strftime("%Y-%m-%dT%H:%M:%S Z"), format))

        def format(self, fmt):
            return self.render("format(\"%s\")" % fmt)

        def calendar(self):
            return self.render("calendar()")

        def fromNow(self):
            return self.render("fromNow()")

注意 *render* 方法并不直接返回字符串而是把它放入了 Jinja2 提供的 *Markup* 对象中。原因是 Jinja2 默认情况下会自动转义，例如，我们的 *<script>* 标签是不可能到达到客户端，因为转义成 *&lt;script&gt;*。把字符串包裹在 *Markup* 对象里就是告诉 Jinja2 这个字符串是不需要转义的。

既然我们有了一个封装的类，我们需要跟 Jinja2 绑定，这样模块就可以使用它(文件 *app/__init__.py*)::

    from momentjs import momentjs
    app.jinja_env.globals['momentjs'] = momentjs

上面的代码就是告诉 Jinja2 导入我们的类作为所有模板的一个全局变量。

现在我们准备修改模版。在我们应用程序中有两个地方显示日期和时间。一个就是用户信息页，那里有最后一次登录时间。对于这个时间戳，我们将会使用 *calendar()* 格式(文件 *app/templates/user.html*)::

    {% if user.last_seen %}
    <p><em>Last seen: {{momentjs(user.last_seen).calendar()}}</em></p>
    {% endif %}

第二个地方就是在 post 子模板，它是被首页，用户信息页以及搜索页调用。在这里我们将会使用 *fromNow()* 格式，因为一篇 blog 的撰写时间和它离现在有多久了是一样重要的。我们需要修改子模板使得所有使用它的页面都有效(文件 *app/templates/post.html*)::

    <p><a href="{{url_for('user', nickname = post.author.nickname)}}">{{post.author.nickname}}</a> said {{momentjs(post.timestamp).fromNow()}}:</p>
    <p><strong>{{post.body}}</strong></p>

做完这些修改后，我们解决了所有我们的时间戳问题。我们不需要对服务器代码做单个的修改！


结束语
-----------

不知不觉中我们已经做了很重要的一步，使得 *microblog* 让国际用户能够根据本地时区看到不同的日期和时间。在下一章中，我们会让国际用户更加高兴，我们让 *microblog* 支持多语言。

如果你想要节省时间的话，你可以下载 `microblog-0.13.zip <https://github.com/miguelgrinberg/microblog/archive/v0.13.zip>`_。

我希望能在下一章继续见到各位！
