---
layout: post
title: "Switching from TDD to BDD with Behat and Symfony2"
date: 2013-08-12 00:37
comments: true
categories: [Symfony, BDD, Behat, Mink]
---
-> {% img fancybox https://dl.dropboxusercontent.com/u/2881323/ftassi.github.io/behavior.jpg %} <-

Recently I started a small extra project with a smart fellow from my [local PHP User Group](http://marche.grusp.org), since the project was quite simple and both of us were willing to learn something new, we decided to give BDD a try. In the PHP world BDD means [Behat](http://behat.org/), [Mink](http://mink.behat.org) and [PHPSpec](http://www.phpspec.net). 

Working on a real project is always different to dealing with documentation, and we had to go through many different issues. Most of the issues related to our old habits with Xunit testing frameworks and some others were due to us getting used to different APIs.

In this post I won't cover all the details about BDD, Behat or PHPSpec, I'd rather describe how I switched from PHPUnit and TDD to BDD (and also show some bits of code).

<!-- more -->

{% pullquote %}
Switching from TDD to BDD is not really about libraries, it is more about switching your mindset from testing to describing. Communication is king here really. It can sound a bit silly, but keep in mind the fact the you're not checking if your code's working, but you're describing how it should work, really IS the key to all this stuff. {" You could indeed do BDD using PHPUnit instead of behat or phpspec, it's not a matter of tools, it's a matter of how you're approaching your code. "}

{% endpullquote %}

That said, tools are important, I found that both behat and phpspec do a great job helping you with switching your mindset from testing to describing. 

The components
--------------

The first step you have to take is to understand which library you have to deal with:

* **Behat** is the real boss: the framework testing. It allows you to write expectations in gherkin language.
* **Mink** is a web acceptance testing library. It allows you to write tests for your web application. You could use it as a stand alone testing library, but using it with behat is way more fun.
* **MinkExtension** is the *glue* beteween Behat and Mink. The extension will configure and boot Mink and you'll be able to use it within Behat *Context classes.
* **Symfony2Extension** gives you access to Symfony kernel from Behat *Context classes and provides a symfony2 session for Mink.

In order to put all these pieces into place you can follow the [official documentation](http://extensions.behat.org/symfony2/), I'm not going to repeat all the steps here.

Organizing your tests
---------------------

{% pullquote %}
Symfony2Extension gives you the ability to have separate feature files for each bundle, or to keep them all together at application level. {" I chose to keep the features at application level "}. Given that features are a description of the application as a whole, it doesn't sound natural to me to have them separated by bundle. Bundle organization is an implementation detail and I don't want this kind of detail exposed in my gherkin features. Of course this is a really personal decision and you should think about how you want to keep your feature files organized. 
{% endpullquote %}

If you, like me, want to keep all the feature files together at application level you have to define it in your configuration file:

```yaml behat.yml
default:
    paths:
        features: features
```

Once you start writing features you also start implementing steps in your *Context classes. 

_In my opinion this is the hardest part_.

Context classes grow really fast; while you're testing your application you're continuously implementing new steps and you always have to keep your feature files and your Context classes easy to read. 

This means that you need to define [macro steps](http://docs.behat.org/guides/2.definitions.html#step-execution-chaining) and you have to organize them into sub contexts in order to keep your code clear. 

You'll also need to get used to jumping from step definition to step implementation which can be a bit confusing at the beginning.

{% pullquote %}
I guess this is not a thing you can learn from documentation, you'll need some experience to get good at this. In this first experience I felt the need to refactor and reorganize my steps implementation a few times, even with a small test suite. The good thing is that every time you move code around you feel that your understanding of the domain becomes deeper. {" Every time a step smelled, it was because I was missing a point in my domain, and every refactoring led me to a better understanding of the application I was building."}
{% endpullquote %}

Launching the tests
-------------------

{% pullquote %}

Right after setting up Behat I started writing features for my application. I didn't write one single feature, I wrote a bunch of them, just to clarify the *big picture* and to help me decide where to start.

Quite immediately after writing the first few scenarios I realized that I didn't want to run them all together, instead I'd have liked to mark some scenarios as "I'm working on this" and other scenarios as "I'll work on this later" or "this is done". 

Tagging was what I was looking for.

{" I tagged all the scenarios as @tbd, the one I was actually working on as @wip and then I defined some profiles in behat.yml "}:

{% endpullquote %}

``` yaml behat.tml
default:
    filters:
        tags:   "~@tbd"
wip:
    filters:
        tags:       "@wip"
```

With this configuration, running:

```bash 
bin/behat 
```

I was able to execute all the scenarios NOT tagged with @tbd
and running:

```bash 
bin/behat --profile wip
```

I was able to run the current "work in progress" scenario.

This is really nice: having different profiles is quite similar to having different suites, and excluding the @tbd tag from the default profile allowed me to execute the whole set of scenarios without having all the *undefined steps* noise from the steps that were not implemented yet. 

{% pullquote %}
This was not my first project guided by tests, but I had never written a whole bunch of functional tests before the implementation. Usually I'd have written the test for the very next feature and then I'd have implemented it. {" Writing in gherkin language helped me to define a much larger set of specifications"}, which is really good because it gave me **the big picture** of what I was doing.

{% endpullquote %}

{% blockquote %}
I think this is one of the big advantages of BDD: it really changes the way you think about your application. Of course, now that I've realized this I could do the same thing using good old PHPUnit, but before this exercise I never focussed on describing a whole set of functionality with a test suite before coding.
{% endblockquote %}

Working with a database
---------------------

{% pullquote %}

Functional testing of your application means dealing with a database and fixtures. The basic rules are always the same: you want to have a clean fresh database at the beginning of each test, then you'll fill it with predefined data and after that you'll make assertions against it. Having a clean starting point for each scenario is the key to avoiding dependencies between them.

The main difference between behat and your good old functional test is that you won't define a set of fixtures in your setUp() method, instead you will define a set of "*Given*" steps.

```gherkin 
Given there is a "user1" User in the database
And there are 2 bookable Courts
```

{" Behat exposes several hooks that you can use to perform actions like dropping and building the database "}, I've used [@BeforeScenario](http://docs.behat.org/guides/3.hooks.html#hooks) for this, with a slightly different implementation of the [LiipFunctionalTestBundle loadFixtures method](https://github.com/liip/LiipFunctionalTestBundle/blob/master/Test/WebTestCase.php#L228). Here is my implementation:

{% endpullquote %}

```php FeaturesContext.php
        public function cleanDatabase()
    {
        $container = $this->kernel->getContainer();
        $registry = $container->get('doctrine');
        $om = $registry->getManager();
        $type = 'ORM';

        $executorClass = 'Doctrine\\Common\\DataFixtures\\Executor\\' . $type . 'Executor';
        $referenceRepository = new ProxyReferenceRepository($om);
        $cacheDriver = $om->getMetadataFactory()->getCacheDriver();

        if ($cacheDriver) {
            $cacheDriver->deleteAll();
        }

        $connection = $om->getConnection();
        if ($connection->getDriver() instanceOf SqliteDriver) {
            $params = $connection->getParams();
            $name = isset($params['path']) ? $params['path'] : $params['dbname'];

            if (!isset(self::$cachedMetadatas)) {
                self::$cachedMetadatas = $om->getMetadataFactory()->getAllMetadata();
            }
            $metadatas = self::$cachedMetadatas;

            // TODO: handle case when using persistent connections. Fail loudly?
            $schemaTool = new SchemaTool($om);
            $schemaTool->dropDatabase($name);
            if (!empty($metadatas)) {
                $schemaTool->createSchema($metadatas);
            }

            $executor = new $executorClass($om);
            $executor->setReferenceRepository($referenceRepository);
        }

        $purgerClass = 'Doctrine\\Common\\DataFixtures\\Purger\\' . $type . 'Purger';
        $purger = new $purgerClass();
        $executor = new $executorClass($om, $purger);
        $executor->setReferenceRepository($referenceRepository);
        $executor->purge();

        return $executor;
    }
```

This implementation is not really optimized for performance but was enough for my project which had a really small database and just a few scenarios. Maybe for a more complex use case you'll need a more efficient way to drop and recreate the database (maybe with some kind of caching of the sqlite file?).

