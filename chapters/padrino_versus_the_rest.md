

- Should care about making your decision about the Framework you want to use?
There's a lot of folks who insist that people who don't know better should just use Rails for the amount of effort that goes into security in Rails, versus having to know about XSS, CSRF, SQL Injection, etc in Sinatra to add Rack Middleware etc to have those protections ...


- Is Rails bad?

Many people nowadays say that Rails is bad. But why? The most told problem with is that it is *bloated* and not easy
to learn in the beginning.


Bloading means, that too many things are loaded during the startup of the application.
Adding a new feature or the installation of a new gem you can increase the memory usage of your application
enourmesly. The easierst way to detect bloats in your system is to grep in your syslog after ressources and check
the memory usage it has. They can jump between a low level and higher level peak. Beside the server you are running
the used server (like puma, webrick, and so on) also influence the performance your application. Also the not cached
usage of ActiveRecord can have is expensive for the Ruby process size. One ActiveRecord usage can easily instantiate
a several k long record which will stay in memory. We often recommend people try Rack::Bug, MemoryLogic or Oink. These are _amazing _time savers and will allow you to inspect how many ActiveRecord instances are being loaded up on any given request. Youe have to know Rails and it's problems to have a feeling where memory leaks can appear and what you can do against them. You can learn more about bloating under this [link](https://blog.engineyard.com/2009/thats-not-a-memory-leak-its-bloat/). When you want to setup a new Rails application, you have to do a lot of installation setup things before you can start working on code => checkout railsapps.github.com/installing-rails...


**Security**: You have Cross-Site Request Forgery,
Mass Assignment, SQL injection, Cross-Site Scripting and HTML njection, and so on. You can read more about the
security issues on the official [rails security guides](http://guides.rubyonrails.org/security.html).


**Asset management**: recompile your assets with Sprockets, precompile with CoffeeScript, add jQuery to the app. It will
rename your assets whenever they change and gives you a slightly different address. j


**Optimization**: Page, action and fragment caching backed by files, memory, MemCacheD or in several other ways. It also has good support for things like ETags and If-Modified-Since via helpers like fresh-when and stale?.


**Support**: Everything Ruby supports, also supports Rails. In the hearts of Rails runs Rack, and Rack supports
Rails, Sinatra, and Padrino. But often even if you’re using Rack middleware, there’s an additional wrapper with convenience functions for Rails, so Rails has the advantage again. For instance, Warden is a nice Rack-based authentication system, but Devise is the same thing nicely packaged for Rails.


- what about other frameworks like node.js


Browser are more and more turning into operating systems. So you should learn Node.js, Backbone, and Coffeescript
because these are languages that can make a paradigm shift in the future. The Firefox Operating System for mobile
devices is fully on this track and the whole operating system is running with JavaScript - which make it better for
developers enhancing the system, write own programs and enhance the basic functions of the system.


- is Padrino better?


- What should you?
Should you learn the dominant paradigm, which will make it easier to get a job, or the new hotness? I’m not worried whether you use Rails, Sinatra, Node.js, http://padrinorb.com%22Padrino, Rulers or something else completely.

But remember that a web app with bad security can compromise everybody else’s security too. When you pick a smaller, simpler framework, know what you’re missing and how to fix it.

Play safe!

