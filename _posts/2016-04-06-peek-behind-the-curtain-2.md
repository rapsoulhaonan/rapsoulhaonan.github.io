---
title: Sinatra 实现原理(二)
subtitle: Helpers 和 Extensions
tags: [Ruby, Sinatra]
---
翻译来自 *Sinatra: Up and Running*  

<!--more-->

#Helpers 和 Extensions

现在我们可以用稍微危险一点的方式使用 Sinatra。虽然使用DSL并不是必须的，可以使用modular风格，但这样做有什么好处呢；毕竟DSL强大方便。但还是有一些DSL原生不支持的事情。

延伸 Sinatra 功能的方式主要有两个：extension methods和helpers。 你其实已经在用extension methods了; 明显的例子就是组成大部分classic程序的route handlers（比如get）。

> 提示
> 
> 当创建extension 和 helper时，best practice是把它们打包在一个module里面并用register方法让Sinatra决定在哪用这些modules作为mixins。程序员何苦为难程序员！

###创建 Sinatra Extensions

我们可以简单的创建一个extension。 假设我们需要发送 GET 和 POST 请求到同一个特定的URL可以被同样的方法处理。 我们这么牛逼，当然要遵循 DRY 原则，不能写两个完全一样的routes的代码。 因此，我们要创建一个extension在处理这个情况。 看看例子3-6.

<sub>Example 3-6. 创建 Sinatra::PostGet extension</sub>
{% highlight ruby %}
require 'sinatra/base'

module Sinatra
  module PostGet
    def post_get(route, &block)
      get(route, &block)
      post(route, &block)
    end
  end

  # register一下
  register PostGet
end
{% endhighlight %}

在“sinatra”文件夹下面简单地创建一个sinatra程序，和你的extension，就叫“post_get.rb”。 例子3-7告诉我们怎么用。

> 提示
> 
> 成功之后你可以试试把`register`语句去掉，会怎么样呢？

<sub>Example 3-7. 使用自定义的 Sinatra::PostGet extension</sub>
{% highlight ruby %}
require 'sinatra'
require './sinatra/post_get'

post_get '/' do
  "Hi #{params[:name]}"
end
{% endhighlight %}

现在我们可以打开telnet看看运行得怎么样。
{% highlight bash %}
$ telnet 0.0.0.0 4567
Trying 0.0.0.0...
Connected to 0.0.0.0.
Escape character is '^]'.
GET / HTTP/1.1
Host: 0.0.0.0

  HTTP/1.1 200 OK
  Content-Type: text/html;charset=utf-8
  Content-Length: 3
  Connection: keep-alive
  Server: thin 1.2.11 codename Bat-Shit Crazy

  Hi

POST / HTTP/1.1
Host: localhost:4567
Content-Length: 7

foo=bar

  HTTP/1.1 200 OK
  Content-Type: text/html;charset=utf-8
  Content-Length: 3
  Connection: keep-alive
  Server: thin 1.2.11 codename Bat-Shit Crazy

  Hi
{% endhighlight %}

成功了吧？看起来还是很厉害的。

###Helpers

Helpers 和 extensions看起来差不多但还是有不同的。 和extension用`register`不同，helpers用`helpers`。╮（￣▽￣）╭ `更重要的是，他们不仅可以用在routes的代码中，还可以用在模版中，在整个程序中提高效率。`

我们来看一个典型的helper方法：生成hyperlinks。 看例子3-8.

<sub>Example 3-8. 一个生成 hyperlinks 的 helper</sub>
{% highlight ruby %}
require 'sinatra/base'

module Sinatra
  module LinkHelper
    def link(name)
      case name
      when :about then '/about'
      when :index then '/index'
      else "/page/#{name}"
    end
  end

  helpers LinkHelper
end
{% endhighlight %}

你只需要`require './sinatra/link_helper'`就可以了。 在Erb里面用的效果看例子3-9.

<sub>Example 3-9. 一个 Erb 模版来测试 module</sub>
{% highlight html %}
<html>
<head>
  <title>Link Helper Test</title>
</head>
<body>
  <nav>
    <ul>
      <li><a href="<%= link(:index) %>">Index</a></li>
      <li><a href="<%= link(:about) %>">About</a></li>
      <li><a href="<%= link(:random) %>">Random</a></li>
    </ul>
  </nav>
</body>
</html>
{% endhighlight %}

这些链接完美地指向`/index`，`/about`，和`/page/random`。

####不在Modules下的Helpers

有时候你就是想简单写一两个方法，另建一个module有点小题大做了。那么你可以用helpers方法。

<sub>Example 3-10. 用 helpers 创建 helper</sub>
{% highlight ruby %}
require 'sinatra'
helpers do
  def link(name)
    case name
    when :about then '/about'
    when :index then '/index'
    else "/page/#{name}"
  end
end

get '/' do
  erb :index
end

get '/index.html' do
  redirect link(:index)
end

__END__

@@index
<a href="<%= link :about %>">about</a>
{% endhighlight %}


###结合 Helpers 和 Extensions

如果你想在extension里面送一个helper，那就可以用一个叫registered的方法，这个方法以app为参数。请看例子3-11。

<sub>Example 3-11. 结合 helpers 和 extensions</sub>
{% highlight ruby %}
require 'sinatra/base'
module MyExtension
  module Helpers
    # helper方法写这里
  end

  # extension方法写这里

  def self.registered(app)
    app.helpers Helpers
  end
end

Sinatra.register MyExtension
{% endhighlight %}

