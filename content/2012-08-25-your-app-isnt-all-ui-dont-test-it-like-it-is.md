+++
title = "Your app isn't all UI, don't test it like it is!"

aliases = ["/2012/08/your-app-isnt-all-ui-dont-test-it-like.html"]
+++

There are two common pieces of advice I give about testing software:
1. Get a walking skeleton up quickly, and drive your development from user-acceptance tests (testing from the UI down), inspired by the fantastic [Growing Object-Oriented Software Guided By Tests](http://www.growing-object-oriented-software.com/).
1. You should have very, very few UI tests[^1].
These two pieces of advice seem to directly contradict each other. They do and they don't.

The idea behind 1 is that you start with a single UI test, and you write many smaller tests as you implement the sub-features required to make the UI test pass, but that having the UI tests gives you the really useful confidence that you get from actually being able to prove your app does all the things you say it does, and discourages you from writing code not required to make the UI test pass.

The idea behind 2 is that UI tests are slow, give you poorly localised feedback, are often flaky, and require writing a lot more test infrastructure and harnesses than smaller tests. So you want to avoid them.

My experience with trying to follow both pieces of advice at the same time has been pretty poor. Driving from UI tests encourages me to write UI tests for _every single thing_. Which is useful, but: a) slow, b) means I'm distracted from getting things done at a lower level, because I'm busy writing UI tests and infrastructure, and c) means I end up with a lot of UI tests.

There's a side-project I've been meaning to write for a while now, and a few times I've started it, gotten bogged down with UI writing and UI testing, gotten bored, and stopped. Yesterday, I started it again, but with a different approach, and it feels much better:

Don't have a UI
---------------

UIs are great; they let people interact with your system. But that's not the thing I want to be testing when I'm implementing logic, or interacting with a data store, or doing any of the many other things which are nothing to do with the UI. When developing core functionality, the UI is really just a way to trigger calls in to your core library. UI tests are great for showing that your UI makes the right calls, and everything is wired up properly, but I definitely don't want them giving me confidence that my core library works! I want much, much smaller tests for that!

So what have I been doing? Well, I've been writing the library first. I know roughly what actions my UI needs to perform in my backend, so I can write code to perform those actions. My code is unit-tested and has some larger tests showing that it interacts with a (fake, in-memory) data store correctly and such, but there are absolutely no UI tests, because there is absolutely no UI! A whole host of things not to distract me!

Now, at some point I'm going to have a problem. One of the drivers for the Walking Skeleton is that integrations are hard, and should be done early so you can find integration problems, and at some point, I'm going to need to integrate my UI (when I write it) with my library, but I'm ok with leaving that to a later date. I like that my UI is largely independent of the implementation. It means my UI tests will really only be testing the UI, and how it integrates with the backend. It means when I'm trying to write the UI using whatever web framework I happen to choose, I can really focus on the UI and the web framework, and not be thinking about a database, or whether my loop can really always terminate. And hopefully, it means that I can write just a few UI tests, which exercise a few useful paths through my system, and show it works, without needing to test every little thing. Because I know every little thing works, I just don't know whether the UI pokes it correctly. So my UI tests can test exactly that.

There are other benefits too. My UI should be more independent of my backend, so I should be able to more easily switch out one or the other with a whole new implementation. Because my library has been designed to be an API for a UI, rather than being the implementation of the UI, opening up that API in other ways should be easier too, as I have a well defined interface. But mostly, it lets me focus on the thing I'm trying to do right now, without being distracted by a largely separate part of the system.

This is something I've been finding works great for me - How do you manage the trade-off between the confidence UI tests give you, and the pain they entail? How do you keep your test-pyramid bottom-heavy?



[^1]: People are often surprised that, given most of my day job involves making UI testing easier, I don't think people should be doing it. I think of it as a necessary evil, which should be minimised. I really liked what someone said at CITCON in London this year, roughly: "Every time someone on my team wants to add a UI test, they have to justify to the team why the thing they're testing can't possibly be tested at a lower level. As a result, we don't have many UI tests, and that's just fine." - sorry I can't attribute the line!
