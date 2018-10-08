---
title: Sinatra 实现原理(四)
subtitle: Dispatching
tags: [Ruby, Sinatra]
---
翻译来自 *Sinatra: Up and Running*  

`这一段暂时超出我的翻译能力，为了完整性先把英文版的给大家看。（需要翻译的同学请尽量留言让我知道是否有必要）`

<!--more-->

#Dispatching

There is, however, one catch: Sinatra relies on the “one instance per request” principle. However, when running as middleware, all requests will use the same instance over and over again. Sinatra performs a clever trick here: instead of executing the logic right away, it duplicates the current instance and hands responsibility on to the duplicate instead. Since instance creation (especially with all the middleware being set up internally) is not free from a performance and resources standpoint, it uses that trick for all requests (even if running as endpoint) by keeping a prototype object around.

Example 3-16 shows the secret sauce in Sinatra’s dispatch activities.

<sub>Example 3-16. The Sinatra dispatch in action</sub>
{% highlight ruby %}
module MySinatra
  class Base
    def self.prototype
      @prototype ||= new
    end

    def self.call(env)
      prototype.call(env)
    end

    def call(env)
      dup.call!(env)
    end

    def call!(env)
      [200, {'Content-Type' => 'text/plain'},
        ['routing logic not implemented']]
    end
  end

  class Application < Base
  end
end
{% endhighlight %}

###Dispatching Redux

This lets us craft some pretty interesting Sinatra applications. This prototype and instance duplication approach means you can safely use call on the current instance and consume the result of another route. If you remember from the earlier discussion on Rack’s methodology, the call method will return an array. The application in Example 3-17 lets you check the status code and headers of other routes. Figure 3-5 shows the output of the inspector application.

<sub>Example 3-17. A reflective route inspector</sub>
{% highlight ruby %}
require 'sinatra'

get '/example' do
  'go to /inspect/example'
end

get '/inspect/*' do
  route  = "/" + params[:splat].first
  data  = call env.merge("PATH_INFO" => route)
  result = "Status: #{data[0]}\n"

  data[1].each do |header, value|
    result << "#{header}: #{value}\n"
  end

  result << "\n"
  data[2].each do |line|
    result << line
  end

  content_type :txt
  result
end
{% endhighlight %}

Now let’s tie everything together with the code in Example 3-18 and create a Sinatra application that acts as middleware.

<sub>Example 3-18. Using Sinatra as middleware in a fictional Rails project</sub>
{% highlight ruby %}
require './sinatra_middleware'
require './config/environment'

use Sinatra::Application
run MyRailsProject::Application
{% endhighlight %}

![img](/img/peek-3-5.png)
<sup>Figure 3-5. Firing another request internally to inspect the response</sup>

###Changing Bindings

To bring the discussion back to where we began, let’s focus on a block passed to get again. How is it that the instance methods are actually available? If you’ve been working with Ruby for a decent length of time, you’ve probably come across instance_eval, which allows you to dynamically change the binding of a block. Example 3-19 demonstrates how this can be used.

<sub>Example 3-19. Toying with instance_eval</sub>
{% highlight ruby %}
$ irb
ruby-1.9.2-p180 > array = ['foo', 'bar']
=> ['foo', 'bar']
ruby-1.9.2-p180 > block = proc { first }
=> #<Proc:0x00000101017c58@(irb):2>
ruby-1.9.2-p180 > block.call
NameError: undefined local variable or method `first' for main:Object
  from (irb):2:in `block in irb_binding'
  from (irb):3:in `call'
  from (irb):3
  from /Users/konstantin/.rvm/rubies/ruby-1.9.2-p180/bin/irb:16:in `<main>'
ruby-1.9.2-p180 > array.instance_eval(&block)
=> "foo"
{% endhighlight %}

This is similar to what Sinatra does. In fact, earlier versions of Sinatra do use instance_eval. However, there is an alternative: dynamically create a method from that block, get the unbound method object for that method, and remove the method immediately. When you want to run the code, bind the method object to the current instance and call it.

This has a few advantages over instance_eval: it results in significantly better performance since the scope change only occurs once as opposed to every request. It also allows the passing of arguments to the block. Moreover, since you can name the method yourself, it results in more readable stack traces. All of this logic is wrapped in Sinatra’s generate_method, which you can examine in Figure 3-6 and Example 3-20.

> Caution
> 
> generate_method is used internally by Sinatra and is not part of the public API. You should not use it directly in your application.

![img](/img/peek-3-6.png)
<sup>Figure 3-6. generate_method and its usage in Sinatra</sup>

<sub>Example 3-20. generate_method from sinatra/base.rb</sub>
{% highlight ruby %}
def generate_method(method_name, &block)
  define_method(method_name, &block)
  method = instance_method method_name
  remove_method method_name
  method
end
{% endhighlight %}
