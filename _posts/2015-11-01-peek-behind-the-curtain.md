---
title: Sinatra 实现原理(一)
subtitle: Application 和 Delegation
header-img: "img/post-bg-sinatra.jpg"
tags: [Ruby, Sinatra]
---
翻译来自 *Sinatra: Up and Running*。  

```在阅读这个专题之前，你应该已经对Sinatra的classic用法风格比较熟悉了。```现在我们更加深入地探讨一下Sinatra是如何运行的。一旦你了解其后台的原理，那么，充分利用可用的API甚至是拓展它就变得非常容易了。为了达到这个目标，我们会研究Sinatra的源代码，看看到底发生了什么。
<!--more-->

> 提示
> 
> Sinatra遵循语义化版本规则，即它一直向下兼容除非主版本升级（版本的第一个数字变大）。因此，在Sinatra 1.2.3下写的程序，也可以运行在1.3.0的版本中。语义化版本控制要求一个官方且全面的API使用规范，在Sinatra里就是README，你可以在[这里](http://www.sinatrarb.com/intro "README")找到。 你可以在[这里](http://semver.org/ "Semantic Versioning")了解更多有关语义化版本。

#Application 与 Delegation

让我们从一个小实验开始。我们应该已经知道网页的参数是通过一个叫params的哈希表传递给routes，并被routes使用。简单的例子就是登录表单一般都会有params[:username] 和 params[:password]。

我们定义的各种routes（get '/home'，post '/login'， 等等）并不是一个函数定义，而是调用了更深层次的Sinatra函数。这些routes的做法，是以block或者叫closure的形式传递给Sinatra去处理和执行。

这样的话，检查这个block被执行的上下文将非常有趣。Ruby里的Block应该与其被定义的作用域具有相同的函数与变量。那么，例子3-1的输出会是什么呢？

<sub>Example 3-1. 检查params作用域的脚本</sub>
{% highlight bash %}
$ irb
ruby-1.9.2-p0 > require 'sinatra'
=> true 
ruby-1.9.2-p0 > get('/') { defined? params }
=> [/^\/$/, [], [], #>Proc:0x00000100aff310@/sinatra-1.2.6/lib/sinatra/base.rb:1152>] 
ruby-1.9.2-p0 > defined? params
=> nil
{% endhighlight %}

如上所示，params只是在get里面才可见。这也揭示了Sinatra运作的秘密就是它处理self变量的方式。

###内部的Self

在Ruby里，没被赋给变量或常量的函数调用，都传给了self。一般来说为了简洁，self都被隐藏了。在例子3-2中，我们调用jobs和self.jobs；输出相同是因为这些调用发生在用一个作用域里面，因而self也是相同的。

<sub>Example 3-2. 展示self的一种用法</sub>
{% highlight bash %}
$ irb
ruby-1.9.2-p0 > jobs
=> #0->irb on main (#<Thread:0x00000100887678>: running) 
ruby-1.9.2-p0 > self.jobs
=> #0->irb on main (#<Thread:0x00000100887678>: running)
{% endhighlight %}

当要通过scope gates的时候，self的唯一性就变得非常重要了；scope gates是指在执行代码中，作用域发生改变的代码片段，往往self也会相应改变。在Ruby里，定义类，定义模块和函数都是scope gates。

例子3-3展示了如何得到self从Sinatra routes内部和外部检测的结果；在Figure3-1可以看到结果。

<sub>Example 3-3. 检测不同作用域下的self</sub>
{% highlight ruby %}
require "sinatra"

outer_self = self
get '/' do
  content_type :txt
  "outer self: #{outer_self}, inner self: #{self}"
end
{% endhighlight %}

> 提示
> 
> 技术上讲，Rubu里的closures会开辟一个新的嵌套的作用域，我们以后再讲。

![result](/img/peek-3-1.png)
<sup>Figure 3-1. 检查不同作用域下的self</sup>

例子3-3告诉我们，routes里面的函数调用实际上都传给了一个Sinatra::Application类的实例。更进一步，如果你刷新那个页面，你会发现object id一直在变化，这说明对于每一个request请求都有不同的实例来处理。

> 提示
>
> 同样值得注意的是外部作用域是main，前面在调用jobs和self.jobs时也一样。

###“get”是哪来的?

现在我们知道routes是把block传递给一个Sinatra::Application的实例处理。那么，route方法到底在哪里实现的呢？我们打开IRB，用Ruby reflection API来大致了解一下例子3-4中展示的类的结构。

<sub>Example 3-4. 使用 reflection</sub>
{% highlight bash %}
[~]$ irb
ruby-1.9.2-p180 > require 'sinatra'
=> true
ruby-1.9.2-p180 > Sinatra::Application.superclass
=> Sinatra::Base
ruby-1.9.2-p180 > Sinatra::Base.superclass
=> Object
ruby-1.9.2-p180 > method(:get)
=> #<Method: Object(Sinatra::Delegator)#get>
ruby-1.9.2-p180 > Sinatra::Delegator.methods(false)
=> [:delegate, :target, :target=]
ruby-1.9.2-p180 > Sinatra::Delegator.target
=> Sinatra::Application
ruby-1.9.2-p180 > Sinatra::Application.method(:get)
=> #<Method: Sinatra::Application(Sinatra::Base).get>
ruby-1.9.2-p180 > _.source_location
=> ["~/gems/sinatra-1.3.0/lib/sinatra/base.rb", 1069]
{% endhighlight %}

有趣的是，像get，post这类的方法实际上被定义了两次。一次是在Sinatra::Delegator，这是一个[mixin]("http://www.tutorialspoint.com/ruby/ruby_modules.htm")。因此这些方法在程序的所有地方都可以使用。这个delegator mixin会直接把同样的函数调用传递给Sinatra::Application，Sinatra::Application从Sinatra::Base继承了这些方法。

> 提示
> 
> Mixins是Ruby从Common Lisp继承的一种技术，这种技术能让你除了选择一个父类以外，还可以加入一些其他的像类一样的的对象到继承链里面。这就让你能够继承现存的类和对象，而不会意外重写其他函数。

我们再深入研究一下Sinatra::Base和Sinatra::Application。例子3-5展示了如何在不用DSL的情况下正确定义routes，以及如果我们在错误的模块下定义routes会发生什么。 

<sub>Example 3-5. 继续使用 reflection</sub>
{% highlight bash %}
[~]$ irb
ruby-1.9.2-p180 > require 'sinatra/base'
=> true
ruby-1.9.2-p180 > get('/') { 'hi' }
NoMethodError: undefined method `get' for main:Object
  from (irb):3
ruby-1.9.2-p180 > Sinatra::Application.get('/') { 'hi' }
=> []
ruby-1.9.2-p180 > Sinatra::Application.run!
== Sinatra/1.3.0 has taken the stage on 4567 for development with backup from Thin
>> Thin web server (v1.2.11 codename Bat-Shit Crazy)
>> Maximum connections set to 1024
>> Listening on 0.0.0.0:4567, CTRL+C to stop
{% endhighlight %}

###探索实现方法

```书上前面内容在介绍了使用顶级DSL的classic风格Sinatra，如果有必要你可以简单从官方文档学习，本文不再赘述。这里讲的是modular style。```  
Sinatra也可以在不用顶级DSL的情况下使用，只需要sinatra/base和一点mixin结构的知识。也意味着顶级DSL不是必须的。

例子3-4和3-5说明了在base.rb里包含对get方法的定义（这个文件也是Sinatra的核心）。在该文件的后面，你可以找到Application和Delegator的实现代码。它们的实现非常简单。而所有Sinatra的主要功能都是在Base类中实现的。 

> 提示
> 
> base.rb现在的开发版本可以在[这里](https://github.com/sinatra/sinatra/blob/master/lib/sinatra/base.rb "sinatra code")找到。


