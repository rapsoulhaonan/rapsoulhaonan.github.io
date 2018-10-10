---
title: Sinatra 实现原理(三)
subtitle: Request，Response 和 Rack
header-img: "img/post-bg-sinatra.jpg"
tags: [Ruby, Sinatra]
---
翻译来自 *Sinatra: Up and Running*  

<!--more-->

#Request 和 Response

更进一步的理解 Sinatra 内部机制，就需要弄懂一个request的流程，包括从处理到传递一个response回客户端。 为此，我们需要了解 Rack 在 Sinatra 中的角色。

###Rack

Rack这个规范不仅只被 Sinatra 使用，Rails，Merb，Ramaze，还有一些别的Ruby项目也同时在使用。 这是一个非常简单的协议，规定一个HTTP服务器（比如Thin）如何与程序对象（比如Sinatra::Application）协作。总之，Rack定义了硬件和软件可以用之相互交流的高等级语言。可以去看看Rack的[主页](http://rack.rubyforge.org "Rack page")。

Rack协议的核心规定程序对象，也就是终端，必须要有`call`方法。服务器，也就是处理者，会调用`call`这个方法，同时还会传递一个参数。这个参数是一个hash，包括所有关于request的信息：HTTP动词（GET，POST），请求路径，还有客户端发来了headers等等。

这个`call`方法需要返回一个包涵三个元素的数组。第一个元素是整型状态码，比如说一个成功的request就会收到一个200状态码，表示没有错误。 第二个是一个hash（或者类似hash），包涵所有response的headers。这里你可以找到客户端是否需要缓存，response的长度以及其他类似的信息。最后一个就是body数组（或者类似于数组），只要有each方法就行。

###不用 Sinatra 的 Sinatra

这样运行一个Sinatra程序而不调用Sinatra的方法是完全可以实现也可以被接受。我们试着把一个简单的sinatra程序放到纯Rack环境下，看例子3-12。

<sub>Example 3-12. Simplified equivalent of a Sinatra application using Rack</sub>
{% highlight ruby %}
module MySinatra
  class Application
    def self.call(env)
      new.call(env)
    end

    def call(env)
      headers = {'Content-Type' => 'text/html'}
      if env['PATH_INFO'] == '/'
        status, body = 200, 'hi'
      else
        status, body = 404, "Sinatra doesn't know this ditty!"
      end
      headers['Content-Length'] = body.length.to_s
      [status, headers, [body]]
    end
  end
end

require 'thin'
Thin::Server.start MySinatra::Application, 4567
{% endhighlight %}

这个例子大致等效于`get('/') { 'hi' }`。 当然这不完全是Sinatra的代码实现，它需要更加全面，要能够处理很多用例，涉及到封装，优化等。 Sinatra会把env这个hash打包到一个方便的对象里面，以request对象的形式供你的代码使用。同样的，response可用于生成body数组。它们都能很轻易地在程序中被调用，看看例子3-13具体怎么做的。 

<sub>Example 3-13. 使用 env, request, 和 response</sub>
{% highlight ruby %}
require 'sinatra'

helpers do
  def assert(condition)
    fail "something is terribly broken" unless condition
  end
end

get '/' do
  assert env['PATH_INFO'] == request.path_info

  final_result = response.finish
  assert Array === final_result
  assert final_result.length == 3
  assert final_result.first == 200

  "everything is fine"
end
{% endhighlight %}

#Rack It Up

如果你再到IRB，输入`Sinatra::Application.new.class`，你会发现这个`new`不会返回一个Sinatra::Application对象，而是一个Rack::MethodOverride对象。 

Rack支持在你的程序之前设置chaining filters和chaining routers。用Rack的术语来说，就叫middleware。middleware同样遵循Rack的规范，有`call`方法并且返回一个上面刚刚提过的数组。它并不是简单的自己生成一个数组，而是调用不同的终端或者middleware的`call`方法。现在这个middleware可以修改request（env hash），也可以修改response，然后在判断是否调用另一个终端或者一系列终端。Sinatra返回Rack::MethodOverride对象而不是Sinatra::Application对象从而保持在这个middleware chaining中。

###Middleware

Rack对于middleware还有额外的规范。middleware由一个[factory](https://en.wikipedia.org/wiki/Factory_(object-oriented_programming) "wiki factory")对象创建。这个对象实现了`new`方法；`new`方法接受至少一个参数，那就是将会被这个middleware打包的终端。最后middleware返回这个打包好的终端。

通常情况下，factory就是一个class，比如Sinatra::ShowException，而这个class的实例就是一个拥有一个固定终端的完整的middleware。我们先把Sinatra放一边，再写一个简单的Rack程序。我们可以用一个`Proc`来代替Sinatra，因为它有`call`方法。我们也会简单写一个middleware来检查路径是否正确。

你的系统应该已经安装了rack这个gem，因为Sinatra依赖这个gem。这个gem包涵一个很好用的工具叫`rackup`，它可以简单的运行一个Rack程序。创建一个文件名叫“config.ru”，输入例子3-14中的内容。之后，你只需要在这个文件夹下运行`rackup -p 4567 -s thin`，就可以在“http://localhost:4567/”看到你的程序了。

<sub>Example 3-14. config.ru 的内容</sub>
{% highlight ruby %}
MyApp = proc do |env|
  [200, {'Content-Type' => 'text/plain'}, ['ok']]
end

class MyMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    if env['PATH_INFO'] == '/'
      @app.call(env)
    else
      [404, {'Content-Type' => 'text/plain'}, ['not ok']]
    end
  end
end

# this is the actual configuration
use MyMiddleware
run MyApp
{% endhighlight %}

###Sinatra 和 Middleware

Rack里的`use`很好用，所以Sinatra也实现了这个方法，用法和Rack完全一下。看看例子3-15是怎么用的。

<sub>Example 3-15. Using use in Sinatra</sub>
{% highlight bash %}
require 'sinatra'
require 'rack'

# A handy middleware that ships with Rack
# and sets the X-Runtime header
use Rack::Runtime

get('/') { 'Hello world!' }
{% endhighlight %}

这些看起来很厉害，但是对我们有什么用呢？这样，你就可以把Sinatra程序用做middleware了啊。

Sinatra::Application这个类就是factory，可以创建配置完整的middleware实例，也就是你的程序实例。当有request进来时，所有的`before`过滤器就会先执行一遍, 如果没有route符合，就把这个request传递给打包好的终端程序。在route处理完或者终端程序处理完之后，`after`过滤器就会被执行一遍。因此，这个Sinatra程序就是一个Rack的middleware。

> 补充
> 
> 这一章较难理解，可以参考一下Rack相关知识。也可以在[这里](http://webapps-for-beginners.rubymonstas.org/rack.html "Webapps For Beginners")找一些简单易懂的补充。
