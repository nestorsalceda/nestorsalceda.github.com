---
layout: post
title: Dependency Injection with Tornado
---

One of the most powerful patterns I use is Dependency Injection.

The main idea behind Dependency Injection is collaboration.  Collaboration among
objects for solving a problem.  Do you remember divide and conquer algorithms?
These algorithms works breaking down a big and complex problem into several
sub-problems which become simpler and simpler and can be solved easily, then the
solutions to the sub-problems must be combined to give a solution for the original
problem.  This is the goal of Dependency Injection, defining the way our objects
are going to be combined for collaborating.

Let's go with an example.  We are going to write a web handler for
inserting some data into a database.

# The Code

{% highlight python %}
from tornado import web

class InsertInDatabaseRequestHandler(web.RequestHandler):
	def initialize(self, mapper):
		self.mapper = mapper

	def post(self):
		entity = self._create_object_from_arguments()
		self.mapper.insert(entity)

	def _create_object_from_arguments(self):
		#...
{% endhighlight %}

This is the web handler.  It receives a POST request with some arguments, builds
an object and pass it to the insert method of mapper instance variable.

{% highlight python %}
from tornado import web

connection = connection_to_database()

application = web.Application([
	web.url('/element', InsertInDatabaseRequestHandler, kwargs={'mapper': EntityMapper(connection)})
])
{% endhighlight %}

This is the application creation.  It maps a URL with a piece of code.

With this code [Tornado][tornado] is going to give us a nice way for performing
Dependency Injection.

# The magic

So we start our cool application and we receive a POST request, next the post method in
InsertInDatabaseRequestHandler is executed, then an entity object is built and passed
to insert method in self.mapper and ... CRASH!!

{% highlight pytb %}
Traceback (most recent call last):
  File "<stdin>", line 2, in post
AttributeError: 'InsertInDatabaseRequestHandler' object has no attribute 'mapper'
{% endhighlight %}

Why does this never happen?

This is because every [web.RequestHandler][request-handler] in [Tornado][tornado] has an [initialize][initialize]
method that is executed as [last step][request-handler-source] in [web.RequestHandler][request-handler]
constructor and receives the kwargs dictionary sent to [web.url][web-url] constructor
as keyed arguments.

In other words, we are able to inject the dependencies of each [web.RequestHandler][request-handler] subclass
through its [initialize][initialize] method.

Quite simple, isn't it?  Were you expecting arcane magic or complex techniques?

# How can this pattern help us to write clean code?

In first place, every class has an unique responsibility, a unique reason to
change.  We are talking about cohesion and orthogonality, because all work related with database
is in the EntityMapper class and the responsibility for dealing with HTTP is in
the RequestHandler.  Perhaps some of you are thinking about the S in
[SOLID][solid] (Single Responsibility Principle).

In second place, I'm able to choose among multiple implementations of a given
dependency at runtime.  If we are a bit smart, we will see the D in
[SOLID][solid] (Dependency Inversion).

Dependency Inversion states that:

* High-level modules should not depend on low-level modules. Both should depend on abstractions.
* Abstractions should not depend upon details. Details should depend upon abstractions.

Do you see the abstraction?  Yes, the [EntityMapper class][datamapper], and perhaps some of
you remember from [P of EAA book][peaa-book].  The main advantage is that I didn't write anything
about the database, I could choose among various strategies: Plain Files, [SQLAlchemy][sqlalchemy], [Redis][redis],
[MongoDB][mongodb] or some mix, have you heard about [Polyglot Persistence][polyglot-persistence]?

Consequently, I can write testable code. But this a topic for another post :)

[tornado]: http://www.tornadoweb.org
[request-handler]: http://www.tornadoweb.org/documentation/web.html#request-handlers
[request-handler-source]: http://www.tornadoweb.org/documentation/_modules/tornado/web.html#RequestHandler
[initialize]: http://www.tornadoweb.org/documentation/web.html#tornado.web.RequestHandler.initialize
[web-url]: http://www.tornadoweb.org/documentation/web.html#tornado.web.URLSpec
[solid]: http://en.wikipedia.org/wiki/SOLID_(object-oriented_design)
[datamapper]: http://martinfowler.com/eaaCatalog/dataMapper.html
[peaa-book]: http://martinfowler.com/books.html#eaa
[sqlalchemy]: http://sqlalchemy.org
[redis]: http://redis.io
[mongodb]: http://mongodb.org
[polyglot-persistence]: http://martinfowler.com/bliki/PolyglotPersistence.html
