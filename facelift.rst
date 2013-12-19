.. _facelift:


换装
=======


简介
--------

如果你一直追随着 *microblog* 应用程序，你可能发现我们并没有在应用程序的外观上花很多的时间。到目前为止，我们使用的模板是基本的，并且没有风格而言。这也是有帮助的，当我们编码的时候，我们不想为编写好看的 HTML 而分心。

这篇文章将会与以前的有所不同，因为写好看的 HTML/CSS 是一个巨大的话题，超出这一系列的预期范围。这里不会有任何 HTML 或 CSS 的细节，我们将只讨论基本的指导方针和思路。


我们该怎么做？
--------------

虽然我们可以认为编码是很难的，但是这些痛苦比不上那些网页设计师，他们必须编写好的并且具有一致性的模板以适应各种浏览器。在如今的社会中，他们不仅仅需要使得设计在常规的浏览器上看起来不错，并且还需要在平板电脑、智能手机上的浏览器上显得好看。

不幸地是，学习 HTML, CSS 以及 Javascript，并且清楚它们在每一种浏览器上的特性是一个深不见底的任务。我们不可能有很多的时间去做。我们只希望少投入些精力让我们的应用程序好看。

因此怎么样才能在这么多限制下完成我们的 *microblog* 界面？


Bootstrap 简介
---------------

我们在 Twitter 里的好朋友发布了一个开源 web 框架，叫做 `Bootstrap <http://twitter.github.com/bootstrap/index.html>`_，它可能就是我们的救命稻草。

Bootstrap 是最常见的网页类型的 CSS 和 Javascript 工具的集合。如果你想要看用这个框架设计的网页，请看这些 `例子 <http://twitter.github.com/bootstrap/getting-started.html#examples>`_。

Bootstrap 擅长如下这些东西:

* 在所有的主流浏览器上看起来一样
* 台式机，平板电脑和手机的屏幕大小不一的处理
* 可定制的布局
* 多风格的导航栏
* 多风格的表单
* 其它很多，很多...


用 Bootstrap 装点 *microblog*
------------------------------

在我们把 Bootstrap 添加到应用程序之前，我们必须安装 Bootstrap CSS，Javascript 以及 图片文件到我们的网页服务器可以找到的地方。

在 Flask 中，*app/static* 文件夹就是这些常规文件所在地。当一个 URL 中有一个 */static* 后缀的话，网页服务器就知道到这个文件夹中寻找文件。

例如，如果我们存储一个名为 *image.png* 文件在 */app/static* 中，我们能够在模板中显示带有如下标签的图片::

	<img src="/static/image.png" />

我们将会根据如下结构来安装 Bootstrap 框架::

	/app
	    /static
	        /css
	            bootstrap.min.css
	            bootstrap-responsive.min.css
	        /img
	            glyphicons-halflings.png
	            glyphicons-halflings-white.png
	        /js
	            bootstrap.min.js

根据 `说明 <http://twitter.github.com/bootstrap/getting-started.html#html-template>`_ ，我们必须在基础模板中的 *head* 部分加入如下内容::

	<!DOCTYPE html>
	<html lang="en">
	  <head>
	    ...
	    <link href="/static/css/bootstrap.min.css" rel="stylesheet" media="screen">
	    <link href="/static/css/bootstrap-responsive.css" rel="stylesheet">
	    <script src="http://code.jquery.com/jquery-latest.js"></script>
	    <script src="/static/js/bootstrap.min.js"></script>
	    <meta name="viewport" content="width=device-width, initial-scale=1.0">
	    ...
	  </head>
	  ...
	</html>

接下来我们需要对模板做的改变有:

* 整个页面的内容使用 `固定的布局 <http://twitter.github.com/bootstrap/scaffolding.html#layouts>`_ 与 `响应功能 <http://twitter.github.com/bootstrap/scaffolding.html#responsive>`_
* 使用 Bootstrap 的 `表单形式 <http://twitter.github.com/bootstrap/base-css.html#forms>`_ 替换所有的表单
* 使用 `导航 <http://twitter.github.com/bootstrap/components.html#navbar>`_ 替换我们的导航栏
* 用 `分页 <http://twitter.github.com/bootstrap/components.html#pagination>`_ 按钮 改变 上一页以及下一页的链接
* 为闪现消息使用 Bootstrap 的 `警告样式 <http://twitter.github.com/bootstrap/components.html#alerts>`_ 
* 使用 `样式图片 <http://twitter.github.com/bootstrap/base-css.html#images>`_ 来表示登录表单中的推荐的 OpenID 提供商
  
我们不会详细解释每一个变化了，因为这些是相当简单。`Bootstrap 官方文档 <http://twitter.github.com/bootstrap/scaffolding.html>`_ 会对大家很有帮助的。


结束语
----------

如果你想要节省时间的话，你可以下载 `microblog-0.12.zip <https://github.com/miguelgrinberg/microblog/archive/v0.12.zip>`_。

我希望能在下一章继续见到各位！