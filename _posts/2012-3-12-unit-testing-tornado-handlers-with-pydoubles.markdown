---
layout: post
title: Unit testing Tornado handlers with pyDoubles
---

A unit test is a *small* and *automated* piece of code written by a developer.
Its purpose is checking a small piece of code works as expected in *isolation*.

So, let's go with an example :)  I'm going to show a little Tornado web application
which does stuff with an external API, [Twitter][twitter],
using an existing library, [tweepy][tweepy], for retrieving mentions.

#Tests are first!

{% highlight python %}
# -*- coding: utf-8 -*-

import httplib

import tweepy
from tornado import web, testing

from hamcrest import *

URL = u'/my_mentions'

CONSUMER_KEY = "xxx"
CONSUMER_SECRET = "xxx"
ACCESS_TOKEN = "xxx"
ACCESS_TOKEN_SECRET = "xxx"


class TestMentionsHandler(testing.AsyncHTTPTestCase):

    def test_get(self):
        #GET /my_mentions
        response = self.fetch(URL)

        assert_that(response.code, is_(httplib.OK))

    def setUp(self):
        auth = tweepy.OAuthHandler(CONSUMER_KEY, CONSUMER_SECRET)
        auth.set_access_token(ACCESS_TOKEN, ACCESS_TOKEN_SECRET)
        self.api = tweepy.API(auth)

        super(TestMentionsHandler, self).setUp()

    def get_app(self):
        return web.Application([
            web.url(URL, MentionsHandler, dict(api=self.api))
        ])

{% endhighlight %}

Tornado has a couple of classes for helping us writing tests. This test case inherits from
[AsyncHTTPTestCase][async-http-test-case], and thus we have to implement the [get\_app][get-app] method.  This
method will be called from setUp method, executed before running each test.  As
we are creating a [web.Application][web-application] object, we are able to use
[dependency injection][dependency-injection-with-tornado] through the
[web.url][web-url] call, passing its dependencies as third parameter, in this case an
[tweepy.API][tweepy-api] object.

So, this test is going to launch a GET request to /my\_mentions URL.  That URL
is mapped to MentionsHandler which is a [web.RequestHandler][web-requesthandler]
and inside its get method we can retrieve mentions, through its
[tweepy.API][tweepy-api] object passed in [web.url][web-url] call, and make stuff with them.

This test is small and automated buuut is far away from being a nice unit test.
Perhaps it could be used as a [Walking Skeleton][walking-skeleton], but is not a
unit test.

#Unit Tests are FIRST!

I wanna mean, [FIRST][unit-tests-are-first]!

* Fast
* Isolated
* Repeatable
* Self-Verifying
* Timely

This test takes in my machine about 3'5 seconds! This is really slow taking
account of a nice velocity is about 100 tests per second. And second, this test is
not repeatable because it needs an internet connection for running it.  And what
about if our API provider is down?

Then, how can we make our test faster and repeatable? We have several choices but,
I would use a technique like [test doubles][test-double-meszaros] And as we are
 writing our code in Python, we have a nice framework for doing it easy,
[pyDoubles][pydoubles] :)

This is an example:

{% highlight python %}
# -*- coding: utf-8 -*-

import httplib

import tweepy
from tornado import web, testing, escape

from pyDoubles.framework import *
from hamcrest import *

URL = u'/my_mentions'


class TestMentionsHandler(testing.AsyncHTTPTestCase):
    def test_get(self):
        when(self.api.mentions).then_return([self._status()])

        response = self.fetch(URL)

        assert_that(response.code, is_(httplib.OK))

    def _status(self):
        return tweepy.Status.parse(tweepy.API(None), escape.json_decode(
          #... JSON that mimics Twitter response
        )

    def setUp(self):
        self.api = stub(tweepy.API(None))

        super(TestMentionsHandler, self).setUp()

    def get_app(self):
        return web.Application([
            web.url(URL, MentionsHandler, dict(api=self.api))
        ])
{% endhighlight %}

I'm replacing the [tweepy.API][tweepy-api] object with an stub.  When mentions
method is called, a canned answer that mimics Twitter response will be returned
instead of requesting it to [Twitter][twitter].

Now this test takes about 0.01 second! This is fast, and is repeatable too!
I can run this on every environment: QA, production, my laptop...  But still
there are something strange in this test, what does it happen if [tweepy.API][tweepy-api]
behaviour is changed in the next version?

Please, remember: Avoid mocking types you can't change!

* Are you going to change third-party code even you have the source code?
* Are you sure the behaviour you are mocking does the same than external
  library?

This is (IMHO) the first D letter of TDD.  The tests are driving your
development, if you listen your tests they are shouting you that there are some
weird stuff in your code.  This is the difference about write tests first or let
tests guide your code!  So what is this test telling us?

This test is telling us about we are passing an instance of
[tweepy.API][tweepy-api] freely around the code.  What about cohesion and
coupling?  How many places I have to check if interface of [tweepy.API][tweepy-api] object
changes?  What about if I change [tweepy][tweepy] to another library which uses [Tornado
ioloop][tornado-ioloop]?

It feels natural we need an abstraction in form of some kind of
MentionsProvider, which simplifies [tweepy.API][tweepy-api] interface.  The only
thing we need are mentions, not friends or followers or direct messages...  Then our
code becomes decoupled from [tweepy.API][tweepy-api] and we are able to write another
provider for [identi.ca](http://identi.ca), for example, and interchange them if
needed.  Anyway this is a topic for another post :)

#pyDoubles love

I started to use [pyDoubles][pydoubles] several months ago.  I loved it
quickly, and I replaced the mocking library I was using for all my Python projects,
including the professional ones.  In fact, [pyDoubles][pydoubles] is my favourite
library for writing test doubles (not only mocks!) when I'm working with Python.

I love Test Driven Development, it feels so natural to me and in
this moment I can't imagine myself writing code without writing their tests
first, it's something like imagining a developer working without an SCM.

As a happy TDD practitioner, I'm pretty used to reread my tests looking for examples,
clarifications, corner cases... And as [PEP-8][pep-8] states, code is read much often
than it's written so that I try to emphasize code legibility.  And, I've found
that I'm able to read [pyDoubles][pydoubles] expectations and asserts from left
to right, like a sentence (or like Hamcrest and [pyHamcrest][pyhamcrest] statements).
That's awesome!  And of course, I can use my [pyHamcrest][pyhamcrest] matchers in order to match
objects with [pyDoubles][pydoubles]. What a handy integration!

I love the fact you can read [pyDoubles unit tests][pydoubles-tests] and its [documentation][pydoubles-documentation].

I like the [Arrange Act Assert pattern][arrange-act-assert-pattern]. It's a really
simple notation and allows me to view at a glance the different steps in my tests.  When
I see that pattern I know that tests are not trying to check several things in
the same test and this is useful for keeping my tests isolated.  I'm able to
use this pattern because [pyDoubles][pydoubles] doesn't rely on (IMHO)
[unnatural metaphors][moq-whats-wrong-with-rrr-model-for-mocking].

I love the [distinction][mocks-arent-stubs] among three types of test doubles: stub, spy and mock.  I'm
able to write more expressive tests because I can refine the kind of
interactions among an object and its neighbors.

Finally, I think that [pyDoubles][pydoubles] is an opinionated piece of
software and if you agree with that opinion is a pleasure work with it.  For
example, other Python Mocking libraries gives you the "feature" of patching
objects.  [pyDoubles][pydoubles] don't.  It forces you to think a bit more about
your design and [SOLID principles][solid] which IMHO is one of the best features
of [pyDoubles][pydoubles].


[twitter]: https://www.twitter.com
[tweepy]: https://github.com/tweepy/tweepy
[dependency-injection-with-tornado]: /2011/11/30/dependency-injection-with-tornado
[async-http-test-case]: http://www.tornadoweb.org/documentation/testing.html#tornado.testing.AsyncHTTPTestCase
[get-app]: http://www.tornadoweb.org/documentation/testing.html#tornado.testing.AsyncHTTPTestCase.get_app
[web-application]: http://www.tornadoweb.org/documentation/web.html#tornado.web.Application
[web-url]: http://www.tornadoweb.org/documentation/web.html#tornado.web.URLSpec
[web-requesthandler]: http://www.tornadoweb.org/documentation/web.html#tornado.web.RequestHandler
[tweepy-api]: http://packages.python.org/tweepy/html/api.html
[walking-skeleton]: http://alistair.cockburn.us/Walking+skeleton
[unit-tests-are-first]: http://pragprog.com/magazines/2012-01/unit-tests-are-first
[test-double-meszaros]: http://xunitpatterns.com/Test%20Double.html
[tornado-ioloop]: http://www.tornadoweb.org/documentation/ioloop.html#module-tornado.ioloop
[pydoubles]: http://www.pydoubles.org/
[pep-8]: http://www.python.org/dev/peps/pep-0008/
[pyhamcrest]: https://github.com/jonreid/PyHamcrest
[arrange-act-assert-pattern]: http://c2.com/cgi/wiki?ArrangeActAssert
[moq-whats-wrong-with-rrr-model-for-mocking]: http://blogs.clariusconsulting.net/kzu/whats-wrong-with-the-recordreplyverify-model-for-mocking-frameworks/
[mocks-arent-stubs]: http://martinfowler.com/articles/mocksArentStubs.html
[pydoubles-documentation]: http://www.pydoubles.org/docs/
[pydoubles-tests]: https://bitbucket.org/carlosble/pydoubles/src/f610a0253abb/pyDoublesTests
[solid]: http://en.wikipedia.org/wiki/SOLID_(object-oriented_design)
