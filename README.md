# ruby-multithreading-with-ruby

My adventure with JRuby with a simple example

## intro

While learning about RabbitMQ and deciding to go with the Ruby bunny client, I started playing around with one of the company's Ruby projects. The project generates 3 pdfs based on the messages that gets routed to it by an exchange. After noticing that usually 2 pdfs gets created pretty fast while the 3rd takes a lot longer, I figured it'd be a good idea to find a way to speed it up. So, the natural thing was to dig into multithreading for Ruby.

A quick Google later, I learned that multithreading is as simple as slapping a Thread.new around a code block like so:

```ruby
Thread.new do
	#Whatever logic you want performed on a new thread
end
```

## wait a sec, those are green

However, after throwing that in, I still see my smaller pdfs get stuck behind the larger one at random. A further consultation of Google revealed that MRI Ruby didn't really have true, complete multithreading. In fact, it didn't have any real multithreading until "recently" (like 1.9.x or something). Prior to that, threading was all done with green threads. Even with 1.9.x, it's still green threads with the exception of certain operations like IO but even that is restricted by the GIL. A good read over this can be found [here](http://www.csinaction.com/2014/10/10/multithreading-in-the-mri-ruby-interpreter/). Reading further on the GIL, it seems that the GIL is core to Ruby and won't be changed anytime soon (more likely never). Thus, I went back to research mode.

## choices

Digging some more brought up a limited amount of choices for concurrency with Ruby. I ended up choosing between using [Celluloid](https://github.com/celluloid/celluloid) or go try out JRuby. After reading a bit on Celluloid, I decided it will probably take a lot longer to implement so I went with JRuby as there're a lot less changes I have to do. In fact, the owner of the RabbitMQ library that the company was using with MRI Ruby, bunny, also had a JRuby implementation called march_hare. From my earlier research with RabbitMQ, it seems he contributed greatly to the other implementations as well. Thus, the conversion from bunny to march_hare was pretty much done within 15 minutes due to how similar the syntax were. I also lucked out that the other gems we used also supported JRuby. In fact, I had more trouble getting JRuby working on my computer than anything else (a thousand blessings to the creator(s) of rvm). Conversions done, I was ready to start work on figuring out how to do multithreading under JRuby.

## enter JRuby

So with JRuby, we gain access to the JVM and can do the ~~nasty, nasty~~awesome, awesome thing where we bring in and mix Java with Ruby code. In fact, all you really need to get started with this unholy abomination of a marriage of 2 codes is to simply require java and have at it:

```ruby
require 'java'
```

Ok, now that we got that out of the way, on to actual multithreading with a code snippet:

```ruby
require 'java'

java_import java.util.concurrent.Executors
executor = Executors.newFixedThreadPool(4)
submit_callable = executor.java_method :submit, [Java::java.util.concurrent.Callable]

tasks = images_list.map do |image|
  submit_callable.call do
    {
      :image_data => retrieve_and_resize_image(image[:uri], image[:link], image[:type]),
      :image_description => image[:description]
    }
  end
end

tasks.each do |t|
  image = t.get
  image_with_description(image[:image_data], image[:image_description])
end

executor.shutdown
```

## the horrors

So now that your eyes are done bleeding from seeing this piece of work and your mind is reeling with disgust, we can get on with the explanations. A simplified explanation of the above code is that I want to create a list of tasks to call a function to go retrieve (download) and resize a collection of images on 4 threads and then add a description to them. This piece makes use of the Java Executor interface, which is decently documented [here](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executor.html).

The thing that probably really stands out is the mess surrounding the ```java_method```. The reason for that is to avoid the warnings you get from reflection if you use ```java_send```. You can also accomplish this with ```java_alias``` but that is not any prettier and you can read about it [here](https://github.com/jruby/jruby/wiki/CallingJavaFromJRuby). The rest should be pretty easy to read or figure out from the docs but here're the basics. In Java, a Future represents the result of an asynchronous computation. Submit submits a value-returning task for execution and returns a Future representing the pending results of the task. The Future's get method will return the task's result upon successful completion. There're a few variations to this but I was ok with this after reaching this point.

## other thoughts

From my experiments on my local machine, I get degraded performance when I create a thread pool larger than the number of cores I have, which makes sense. I also experimented with the different types of thread pools you can create. If a project has unlimited resources, I'd go with the newCachedThreadPool (documentation for this and other types [here](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executors.html)). However, if your resources are limited and you know the type of machine your code will go on, it seems to be better to go the fixed size route. In other words, you can get maximum utilization out of your Heroku dynos as your logs are peppered with warnings stating you are going over 100% of your resources available. On production instance of Heroku, we had to 2x the dyno but decrease the number of workers with an overall cost savings with this implementation. Locally, your machine's fans will probably start to spin faster as your machine gets a little hot to the touch like mine did.
