#The Fast Web

##Background

Web Development, there is a common belief that there is more to be gained by creating abstractions and frameworks, using programming langauges that are `easier` to work with, that lower the barrier for entry, and make it faster to get started actually writing the logic for your application. Rather than writing the fastest application you can for your use case.

For example Java + SpringMVC + Hibernate connected to Postgres, running on Tomcat behind Apache or Nginx, there are lots of examples of this level of abstraction in other languages.

Even the most low level Java web framework tends to build on something like Jetty to handle TCP connections, and has abstraction for handling all types of HTTP Requests and Responses.

Part of the problem is really well described in [this](https://www.youtube.com/watch?v=kZRE7HIO3vk) talk by Casey Muratori, I encourage people to watch it.

I am not suggesting that I write an OS to host a Web Application, or that I write my own database, or database drivers. Until and if Casey's idea becomes reality you need to assume that people writing such low level things as databases and database drivers are writing the best, fastest code they can write, anything more and you would never finish any project.

Instead this project is an attempt to see how far it is reasonable to go towards removing code from the process. To start, the biggest gains would be:
* Not using a framework or library to accept TCP connections, this would be done using the lowest level available to us, OS sockets.
* Not using database abstractions such as ORM's, using the lowest level possible, database drivers supplied from the people that write the database.

At the most basic level, on linux, you can open a TCP socket, recieve HTTP requests and return responses in around 80-100 lines of C.

Web Frameworks have a lot of code, hundreds of thousands, potentially a million plus and there is a lot of stuff that a web framework does for you, but they are heavily generic, to allow the creation of any web application.

Writing a request handler is a simple case of parsing the HTTP request, getting the URL, paramters and body, then sending an HTTP response.

If you were to deal with all the different requests with just functions, only extracting from the request what you need and returning in the response what was required, extracting common functions for your specific application, rather than building abstractions for any application, would this be a better approach?

This is what I am going to try and figure out with this project. By building a web application from basic principles.

##Early Descisions

###Language
I will be writing this to be as fast as possible, so I will be using C++ written in a quite non-C++ way, what I mean by this is that I will almost be writing C, but using some of the C++ standard library to make the code a bit nicer, better structs, vectors, smart pointers etc.

There are a couple of languages coming that would be really cool write this in, Zig and Odin, but they aren't production ready. Also Jai, not released at all yet, looks really good but I feel the standard library would be way too focused on gaming to be useful.

###Thread per request vs event loop
This is an important decision early on.

One thread per request is conceptually easier to understand, request comes in, thread is created, all handling of the request is done on that thread, it can block and wait on I/O and further network requests, the only interaction between threads is if you have some kind of in-memory session storage, that remebers things about previous and current requests, that needs to be carefully handled with locking and mutexes.

An Event Loop is a essentially a queue of small function calls, you have one thread handling the incoming requests, one thread handing the queue and a couple more threads handling specific long running blocking tasks. There is a lot more potential for races and general multi-threaded issues.

Event loops, when done well, can handle a near infinite number of concurrent requests, because it doesn't actually handle them, it only deals with one thing at a time.

However, I feel that in general, unless you are expecting millions of requests a day, the complexity and mental overhead required to write a web service keeping in mind the event loop is not going to be beneficial. Just something as simple as writing to disk or doing a datbase query has to be wrapped in some kind of callback, or promise type system and run on a different thread, because you don't want to be I/O blocking on the event loop thread. This can work in something like NodeJS, because the entire Node standard library is designed in this way. Vertx is a good example of a Java project that gets amazing performance from an Event driven architecture, but at the expense of not knowing when things are actually being called in your code.

So I will be dealing with the, potential, performance hit of creating one thread per request. If you are dealing with a high load in this scenario, you can block once you hit some max level, Tomcat, for example, only allows 500 simultaneous requests by default, but if your application can respond fast enough for each request, this isn't really an issue and the number can be higher, any pauses will be negligible. If you are in the envious position of running out of resources because of load, horizontal scaling is going to be the best option.

##Deployment

Deployment should be trivial, 