.. _ajax:

Ajax
=======

这将是国际化和本地化的最后一篇文章，我们将会尽所能使得 *microblog* 应用程序对非英语用户可用和更加友好。

不知道大家平时有没有见过网站上有一个 “翻译” 的链接，点击后会把翻译后的内容在旁边显示给用户，这些链接触发一个实时自动翻译的内容。谷歌显示这个 “翻译” 链接是为了能够显示外国语言的搜索结果。Facebook 显示它为了能够翻译 blog 内容。今天我们将要添加同样的功能的 “翻译” 链接到我们的 *microblog*！


客户端 VS 服务器端
--------------------

在传统的沿用至今的服务器端的模型中，有一个客户端(用户的浏览器)发送请求到我们的服务器上。一个请求能够简单地请求一个页面，像当你点击 “你的信息” 链接，或者它能够让我们执行一个动作，像当用户编辑他的或者她的用户信息并且点击提交的按钮。在这两种类型的请求中服务器通过发送一个新的网页到客户端，直接或通过发出一个重定向的请求来完成这次请求。客户端接着使用新页代替目前的页面。这个循环就会重复只要用户停留在我们的网页上。我们叫这种模式为服务器端，因为服务器做了所有的工作而客户端只是在它们收到页面的时候显示出来。

在客户端模式中，我们有一个网页浏览器，再次发送请求到服务器上。服务器会像服务器端模式一样回应一个网页，但是不是所有的页面数据都是 HTML，同样也有代码，典型的就是用 Javascript 编写的。一旦客户端接收到页面会把它显示出来并且会执行携带的代码。从此，你有一个活跃的客户端，可以做自己的工作，没有接触外面的服务器。在严格的客户端，整个应用程序被下载到客户端在初始页面请求中，然后应用程序运行在客户端上不会刷新页面，只有向服务器获取或存储数据。这种类型的应用称为 `单页应用 <http://en.wikipedia.org/wiki/Single-page_application>`_ 或者 SPAs。

大多数应用是这两种模式的结合体。我们的 *microblog* 应用程序是一个完全的服务器端应用，但是现在我们想要添加些客户端行为。为了实现实时翻译用户的 blog 内容，客户端浏览器将会发送一个请求到服务器，但是服务器将会回应一个翻译文本而且不需要页面刷新。客户端将会动态地插入翻译到当前页面。这种技术叫做 `Ajax <http://en.wikipedia.org/wiki/Ajax_(programming)>`_，这是 Asynchronous Javascript and XML 的简称。


翻译用户生成内容
-------------------

多亏了 Flask-Babel 我们现在比较好的支持了多语言。假设我们能找到愿意帮助我们的翻译器，我们可以在尽可能多的语言中发布我们的应用程序。

但是还有一个遗漏问题。因为有很多各种语言的用户使用系统，那么用户发布的 blog 内容的语言也是多种的。可能不是本语种的用户不知道内容的含义，如果我们能够提供一种自动翻译的服务这种会不会更好？

这是一个用 Ajax 服务来实现的理想的功能。我们的首页可以显示很多的 blog，它们中的一些可能是不同语言的。如果我们使用传统的服务器端模式来实现翻译的话，一个翻译的请求可能会让原始页面被新的只翻译了选择的 blog 的页面替换。在用户读完翻译后，我们必须点击回退键去获取 blog 列表。事实上请求一个翻译并不是需要更新全部的页面，这个功能让应用程序更好，因为翻译的文本是动态地插入到原始的文本下面，其他的内容不会发生变化。因此，今天我们将会实现我们的 Ajax 服务。

实现实时翻译需要几个步骤。首先，我们需要确定要翻译的文本的原语言类型。一旦我们知道原语言类型我们也就知道需不需要对一个给定的用户翻译，因为我们也知道这个用户选择的语言类型。当翻译被提供，用户希望看到它的时候，将会调用 Ajax 服务。最后一步就是客户端的 javascript 代码将会动态地把翻译文本插入到页面中。


确定 blog 语言
------------------

我们首先的问题就是确定 blog 撰写的语言。这不是一门精确的科学，它不会总是能够检测的语言的类型，所以我们只能尽最大努力去做。我们将会使用 `guess-language <http://code.google.com/p/guess-language/>`_ Python 模块。因此，请安装这个模块。针对 Linux 和 Mac OS X 用户::

    flask/bin/pip install guess-language

针对 Windows 用户::

    flask\Scripts\pip install guess-language

有了这个模块，我们将会扫描每一篇 blog 的内容试着猜测它的语言种类。因为我们不想一遍一遍地扫描同一篇 blog，我们将会针对每一篇 blog 只做一次，当用户提交 blog 的时候就去扫描。我们将会把每一篇 blog 的语言种类存储在数据库中。

因此让我们开始在我们的 Post 表中添加一个 *language* 字段::

    class Post(db.Model):
        __searchable__ = ['body']

        id = db.Column(db.Integer, primary_key = True)
        body = db.Column(db.String(140))
        timestamp = db.Column(db.DateTime)
        user_id = db.Column(db.Integer, db.ForeignKey('user.id'))
        language = db.Column(db.String(5))

每一次修改数据库，我们都需要做一次迁移::

    $ ./db_migrate.py
    New migration saved as microblog/db_repository/versions/005_migration.py
    Current database version: 5

现在我们已经在数据库中有了存储 blog 内容语言类型的地方，因此让我们检测每一个 blog 语言种类::

    from guess_language import guessLanguage

    @app.route('/', methods = ['GET', 'POST'])
    @app.route('/index', methods = ['GET', 'POST'])
    @app.route('/index/<int:page>', methods = ['GET', 'POST'])
    @login_required
    def index(page = 1):
        form = PostForm()
        if form.validate_on_submit():
            language = guessLanguage(form.post.data)
            if language == 'UNKNOWN' or len(language) > 5:
                language = ''
            post = Post(body = form.post.data, 
                timestamp = datetime.utcnow(), 
                author = g.user, 
                language = language)
            db.session.add(post)
            db.session.commit()
            flash(gettext('Your post is now live!'))
            return redirect(url_for('index'))
        posts = g.user.followed_posts().paginate(page, POSTS_PER_PAGE, False)
        return render_template('index.html',
            title = 'Home',
            form = form,
            posts = posts)

如果语言猜测不能工作或者返回一个非预期的结果，我们会在数据库中存储一个空的字符串。


显示 “翻译” 链接
-------------------

接下来一步就是在 blog 旁边显示 “翻译” 链接(文件 *app/templates/posts.html*)::

    {% if post.language != None and post.language != '' and post.language != g.locale %}
    <div><a href="#">{{ _('Translate') }}</a></div>
    {% endif %}

这个链接需要我们添加一个新的翻译文本， “翻译”('Translate') 是需要被包含在翻译文件里面，这里需要执行前面一章介绍的更新翻译文本的流程。

我们现在还不清楚如何触发这个翻译，因此现在链接不会做任何事情。


翻译服务
------------

在我们的应用能够使用实时翻译之前，我们需要找到一个可用的服务。

现在有很多可用的翻译服务，但是很多是需要收费的。

两个主流的翻译服务是 `Google Translate <https://developers.google.com/translate/>`_ 和 `Microsoft Translator <http://www.microsofttranslator.com/dev/>`_。两者都是有偿服务，但微软提供的是免费的入门级的 API。在过去，谷歌提供了一个免费的翻译服务，但已不存在。这使我们很容易选择翻译服务。


使用 Microsoft Translator 服务
--------------------------------

为了使用 Microsoft Translator，这里有一些流程需要完成:

* 应用的开发者需要在 Azure Marketplace 上注册 `Microsoft Translator app <https://datamarket.azure.com/dataset/1899a118-d202-492c-aa16-ba21c33c06cb>`_。这里可以选择服务级别(免费的选项在最下面)。
* 接着开发者需要 `注册应用 <https://datamarket.azure.com/developer/applications/>`_。注册应用将会获得客户端 ID 以及客户端密钥代码，这些用于发送请求的一部分。

一旦注册部分完成，接下来处理请求翻译的步骤如下:

* `获取一个访问令牌 <http://msdn.microsoft.com/en-us/library/hh454950.aspx>`_，需要传入客户端 ID 和客户端密钥。
* 调用需要的翻译服务，`Ajax <http://msdn.microsoft.com/en-us/library/ff512404.aspx>`_，`HTTP <http://msdn.microsoft.com/en-us/library/ff512419.aspx>`_ 或者 `SOAP <http://msdn.microsoft.com/en-us/library/ff512435.aspx>`_，提供访问令牌和要翻译的文本。

这听起来很复杂，因此如果不需要关注细节的话，这里有一个做了很多“脏”工作并且把文本翻译成别的语言的函数(文件 *app/translate.py*)::

    import urllib, httplib
    import json
    from flask.ext.babel import gettext
    from config import MS_TRANSLATOR_CLIENT_ID, MS_TRANSLATOR_CLIENT_SECRET

    def microsoft_translate(text, sourceLang, destLang):
        if MS_TRANSLATOR_CLIENT_ID == "" or MS_TRANSLATOR_CLIENT_SECRET == "":
            return gettext('Error: translation service not configured.')
        try:
            # get access token
            params = urllib.urlencode({
                'client_id': MS_TRANSLATOR_CLIENT_ID,
                'client_secret': MS_TRANSLATOR_CLIENT_SECRET,
                'scope': 'http://api.microsofttranslator.com', 
                'grant_type': 'client_credentials'
            })
            conn = httplib.HTTPSConnection("datamarket.accesscontrol.windows.net")
            conn.request("POST", "/v2/OAuth2-13", params)
            response = json.loads (conn.getresponse().read())
            token = response[u'access_token']

            # translate
            conn = httplib.HTTPConnection('api.microsofttranslator.com')
            params = {
                'appId': 'Bearer ' + token,
                'from': sourceLang,
                'to': destLang,
                'text': text.encode("utf-8")
            }
            conn.request("GET", '/V2/Ajax.svc/Translate?' + urllib.urlencode(params))
            response = json.loads("{\"response\":" + conn.getresponse().read().decode('utf-8-sig') + "}")
            return response["response"]
        except:
            return gettext('Error: Unexpected error.')

这个函数从我们的配置文件中导入了两个新值，id 和密钥代码(文件 *config.py*)::

    # microsoft translation service
    MS_TRANSLATOR_CLIENT_ID = '' # enter your MS translator app id here
    MS_TRANSLATOR_CLIENT_SECRET = '' # enter your MS translator app secret here

上面的 ID 和密钥代码是需要自己去申请，步骤上面已经讲了。即使你只希望测试应用程序，你也能免费地注册这项服务。

因为我们又添加了新的文本，这些也是需要翻译的，请重新运行 *tr_update.py*，*poedit* 和 *tr_compile.py* 来更新翻译的文件。


让我们翻译一些文本
--------------------

因此我们该怎样使用翻译服务了？这实际上很简单。这是例子::

    $ flask/bin/python
    Python 2.6.8 (unknown, Jun  9 2012, 11:30:32)
    >>> from app import translate
    >>> translate.microsoft_translate('Hi, how are you today?', 'en', 'es')
    u'¿Hola, cómo estás hoy?'


服务器上的 Ajax
------------------

现在我们可以在多种语言之间翻译文本，因此我们准备把这个功能整合到我们应用程序中。

当用户点击 blog 旁的 “翻译” 链接的时候，会有一个 Ajax 调用发送到我们服务器上。我们将看看这个调用是如何生产的， 现在让我们集中精力实现服务器端的 Ajax 调用。

服务器上的 Ajax 服务像一个常规的视图函数，不同的是不返回一个 HTML 页面或者重定向，它返回的是数据，典型的格式化成 `XML <http://en.wikipedia.org/wiki/XML>`_ 或者 `JSON <http://en.wikipedia.org/wiki/JSON>`_。因为 JSON 对 Javascript 比较友好，我们将使用这种格式(文件 *app/views.py*)::

    from flask import jsonify
    from translate import microsoft_translate

    @app.route('/translate', methods = ['POST'])
    @login_required
    def translate():
        return jsonify({ 
            'text': microsoft_translate(
                request.form['text'], 
                request.form['sourceLang'], 
                request.form['destLang']) })

这里没有多少新内容。这个路由处理一个携带要翻译的文本以及原语言类型和要翻译的语言类型的 POST 请求。因为这是个 POST 请求，我们获取的是输入到 HTML 表单中的数据，请直接使用 *request.form* 字典。我们用这些数据调用我们的一个翻译函数，一旦我们获取翻译的文本就用 Flask 的 *jsonify* 函数把它转换成 JSON。客户端看到的这个请求响应的数据类似这个格式::

    { "text": "<translated text goes here>" }


客户端上的 Ajax
-------------------

现在我们需要从网页浏览器上调用 Ajax 视图函数，因为我们需要回到 *post.html* 子模板来完成我们最后的工作。

首先我们需要在模版中加入一个有唯一 id 的 *span* 元素，以便我们在 `DOM <http://en.wikipedia.org/wiki/Document_Object_Model>`_ 中可以找到它并且替换成翻译的文本(文件 *app/templates/post.html*)::

    <p><strong><span id="post{{post.id}}">{{post.body}}</span></strong></p>

同样，我们需要给一个 “翻译” 链接一个唯一的 id，以保证一旦翻译显示我们能隐藏这个链接::

    <div><span id="translation{{post.id}}"><a href="#">{{ _('Translate') }}</a></span></div>

为了做出一个漂亮的并且对用户友好的功能，我们将会加入一个动态的图片，开始的时候是隐藏的，仅仅出现当翻译服务运行在服务器上，同样也有唯一的 id::

    <img id="loading{{post.id}}" style="display: none" src="/static/img/loading.gif">

目前我们有一个名为 *post<id>* 的元素，它包含要翻译的文本，还有一个名为 *translation<id>* 的元素，它包含一个 “翻译” 链接但是不久就会被翻译后的文本代替，也有一个 id 为 *loading<id>* 的图片，它将会在翻译服务工作的时候显示。

现在我们需要在 “链接” 链接点击的时候触发 Ajax。与直接从链接上触发调用相反，我们将会创建一个 Javascript 函数，这个函数做了所有工作，因为我们有一些事情在那里做并且不希望在每个模板中重复代码。让我们添加一个对这个函数的调用当 “翻译” 链接被点击的时候::

    <a href="javascript:translate('{{post.language}}', '{{g.locale}}', '#post{{post.id}}', '#translation{{post.id}}', '#loading{{post.id}}');">{{ _('Translate') }}</a>

变量看起来有些多，但是函数调用很简单。假设有一篇 id 为 23，使用西班牙语写的 blog，用户想要翻译成英语。这个函数的调用如下::

    translate('es', 'en', '#post23', '#translation23', '#loading23')

最后我们需要实现的 *translate()*，我们将不会在 *post.html* 子模板中编写这个函数，因为每一篇 blog 内容会有些重复。我们将会在基础模版中实现这个函数，下面就是这个函数(文件 *app/templates/base.html*)::

    <script>
    function translate(sourceLang, destLang, sourceId, destId, loadingId) {
        $(destId).hide();
        $(loadingId).show();
        $.post('/translate', {
            text: $(sourceId).text(),
            sourceLang: sourceLang,
            destLang: destLang
        }).done(function(translated) {
            $(destId).text(translated['text'])
            $(loadingId).hide();
            $(destId).show();
        }).fail(function() {
            $(destId).text("{{ _('Error: Could not contact server.') }}");
            $(loadingId).hide();
            $(destId).show();
        });
    }
    </script>

这段代码依赖于 `jQuery <http://jquery.com/>`_，需要详细了解上述几个函数的话，请查看 `jQuery <http://jquery.com/>`_。


结束语
---------

近来当使用 Flask-WhooshAlchemy 为全文搜索的时候，会有一些数据库的警告。在下一章的时候，我们针对这个问题来讲讲 Flask 应用程序的调试技术。

如果你想要节省时间的话，你可以下载 `microblog-0.15.zip <https://github.com/miguelgrinberg/microblog/archive/v0.15.zip>`_。

我希望能在下一章继续见到各位！