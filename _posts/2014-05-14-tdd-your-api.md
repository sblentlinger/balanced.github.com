---
layout: post
author: Steve Klabnik
author_image: /img/authors/steve_klabnik.png
title: "TDD your API"
image: /img/blogimages/5_14_2014_image_600x424.jpg
cover_image: /img/blogimages/5_14_2014_image_1020x340.jpg
tags:
- balanced
- api
- engineering
---

On August 16, 2012, a process kicked off at Balanced. It still isn't finished,
but this week was a major milestone, and I'd like to share our progress with
you.

That process is a new development methodology for APIs. Rather than trying to
give it some grand name, I'm calling this "TDD for APIs."

## How do you TDD an API?

If you're not familiar, here's the basic outline of Test Driven Development:

1. Write a test for some behavior you'd like to introduce into your system.
2. Run your test suite, and make sure that test fails.
3. Write the simplest code that implements the behavior.
4. Run your test suite, and make sure that test passes.
5. Refactor, because the simplest code often has undesirable properties.
6. Commit, and GOTO 1.

To say that there's a large amount of literature on the benefits of this
approach would be an understatement. I just want to focus on one of the
properties of this approach: verifying behavior. TDD verifies behavior in a
few ways. The first is in step two, with the failing test. This failing test
is important because it verifies that our test does test for a new behavior.
If the test was passing at this point, we probably have a bug in our test. The
second way that behavior is verified in TDD is step four, when the test passes.
This verifies that our new behavior satisfies the contract we've laid out in
the test, which is incredibly important! Finally, when software is changed, we
use TDD to verify that only the behavior we were interested in changing has
changed. We don't want our good change to break anything else.

So how is this different when applied to the API context? Here's how you to
TDD for an API:

1. Create a separate project for your API tests.
2. Write a test for some behavior you'd like to introduce into your system.
   Because this test suite is in a different project, this will be an
   integration-style test, which makes HTTP calls against your customer-facing
   test environment.
3. Run your test suite, and make sure that test fails.
4. Implement this behavior in your system, and push that code to your test
   environment.
5. Run your test suite, and make sure that test passes.
6. Refactor any of your test code that may have gotten messy.
7. Commit, and GOTO 1.

There's another aspect which is important, though. This project should be at
least partially public, and included with all of your other documentation.
Because Balanced is an [open company](http://www.opencompany.org/), ours is
100% public, but if you aren't doing new feature development in the open the
way we are, just have the latest passing suite public. This aspect is important
because, in this model, these API tests become the canonical source of truth
for what your API provides to customers. Your customers can trust them because
they're automatic, and open. Does the API work? Check the build status.

[Push to card](https://www.balancedpayments.com/push-to-card) was the first
real feature we've built in this way. It's been a long time coming, though.
Let's take a journey.

## Origins

[Humble
beginnings](https://github.com/balanced/balanced-api/commit/b8f63e815bbb1d6e097e8b065e64449e1b6a5463).
From the start, there was an idea that _something_ needed to be built to
handle some API-related things. With this dummy commit, the first question
kicked off: [Markdown or reStructured
Text](https://github.com/balanced/balanced-api/issues/1).

There were three big questions to tackle:

1. How can we validate that our API is working as intended?
2. Can we generate documentation from this?
3. How would all this be written, tools-wise?

While this was being decided, at least Issues provided a good forum to discuss
things. From [the first version of the
README](https://github.com/balanced/balanced-api/pull/4):

> The primary goal of this repo is to create more openness behind the decisions
> driving the designs and functionality of the Balanced API. We reached out to
> existing and potential customers when designing the API, but that was a
> limited set of people we already knew. We've received tremendous growth in
> the last few months, and our new customers have great feedback or at least
> want to understand the reasoning behind the original decisions.

## An initial solution

It was settled that reStructured Text was the answer. [Here's what they looked
like](https://github.com/balanced/balanced-api/commit/a71a415311929c88d8e8c976d6199b493953800c):

```
===============
customers.create
===============

:uri: /v1/marketplaces/(marketplace:marketplace)/customers
:methods: POST
```

This worked okay, but there were a few problems: there was no way for scenarios
to refer to each other, and while the tests _ran_ in an automated fashion,
they'd require a human to check for passing or failing. This was because of
things like timestamps which would be a fraction of a section off, or a
generated ID number that'd appear in the output which couldn't be predicted
from the start.

reStructured Text wasn't really designed to do this, anyway. A new approach was
sought.

## The YAML year

The next iteration of this idea used YAML instead of rST. The YAML tests looked
like this:

```
scenarios:
  - name: customer_create
    request:
      method: POST
      href: /customers
      body:
        name: some name
    response:
      status_code: 201
      body: |
        { "$ref": "../responses/customers.json" }
```

You can explore the full suite
[here](https://github.com/balanced/balanced-api/tree/revision0).

It's almost a serialized HTTP request. Since they have names, they can refer to
each other, and you can do things like interpolate variables, use a regular
expression to only match the parts you care about, and make scenarios depend on
each other, for re-usability.

All wasn't well in this world, though. Nobody actually wants to write YAML.
Nobody actually wants to read YAML.  Tests that refer to each other help
reusability, but harm understanding. Since this was entirely home-grown, people
didn't really know how it worked. This also meant that there was a large amount
of work to get up and running.

Time to throw it all out and do something again!

At this point, it was clear that building something totally custom wasn't the
right answer. The maintenance costs are just too high. And other people
build APIs, so they _must_ do something like this, right? Well, while there
are some tools for doing things like this, none of them are particuarly great.
They were all missing at least one thing that we considered vital. One of the
bigger issues is that many of these tools assume that you're simply doing 'Rails
style REST,' and not using hypermedia. Or they require you to write a WSDL or
WADL or use RAML.

## Eat your cucumbers!

Replacing this system was my first task here. [I decided to use
Cucumber](https://github.com/balanced/balanced-api/pull/431). The Cucumber
tests look like this:

```
Feature: Customers

  Scenario: Creating a customer
    When I POST to /customers with the JSON API body:
      """
      {
        "customers": [{
          "name": "Balanced Testing"
        }]
      }
      """
    Then I should get a 201 Created status code
    And the response is valid according to the "customers" schema
    And the fields on this customer match:
      """
      {
        "name": "Balanced Testing"
      }
      """
```

I'm often skeptical of Cucumber, and it's often mis-used, but I think this is
a pretty great use case. It's language agnostic, which is important to us, and
it's got lots of room for explanations and clarifications.

[Here](https://github.com/balanced/balanced-api/pull/580) are the specs for
push-to-card. By using this approach, we caught problems before the
feature was even finished. The initial implementation contained a bug with
a typo in one of the response keys. There was [some
confusion](https://github.com/balanced/balanced-api/issues/607) in one corner
of the spec, and while some of the discussion happned in person, the ways in
which the spec failed while the feature was being built out helped [make sure
that everyone was on the same
page.](https://github.com/balanced/balanced-api/issues/607#issuecomment-43114594)

## Getting outside feedback

Having a public repository to talk about all of this has been very valuable.
Even in the intial implementation, I was able to solicit feedback from [the
authors](https://github.com/balanced/balanced-api/pull/431#issuecomment-29705876)
of
Cucumber](https://github.com/balanced/balanced-api/pull/431#issuecomment-29884794)
and some [skeptical feedback from
friends](https://github.com/balanced/balanced-api/pull/431#issuecomment-29706071).
It's much harder to ask for feedback when you can't share the code. We try to be
mostly open source anyway, but if we weren't, I wouldn't have to worry that I'd
be leaking sensitive information, because the test repository is 100% public.

We've also been able to get feedback on new features too. When Jon Matonis
[criticized Balanced for not supporting
Bitcoin](http://www.forbes.com/sites/jonmatonis/2012/11/26/payments-startup-balanced-innovates-in-wrong-direction/),
we just [opened an issue](https://github.com/balanced/balanced-api/issues/204).
There's a ton of discussion there, and we were able to explain some of our
concerns, try to find solutions, and at least acknowledge the request in some
way. Over a year later, this has materialized into our [partnership with
Coinbase](http://blog.balancedpayments.com/bitcoin/), and I'll be able to close
that issue soon.

## Going forward

Now that we have a testable validation that our API is working as intended, we
can do all kinds of things. We'd like to integrate this suite into our more
general continuous integration suite. We'd like to run this test every hour, or
every three hours, which can be a form of alerting against 
[regressions](https://github.com/balanced/balanced-api/issues/591).

In order to do all that, we need to [fix our
build](https://travis-ci.org/balanced/balanced-api/builds/25288588). That's
what happens when you don't run the tests automatically: regressions can creep
in and you won't realize it. These failures are all related to [a
bug](https://github.com/balanced/balanced-api/issues/591) which only happens
with a brand new marketplace, and doesn't strictly affect customer behavior. I
wanted the fix to roll out before this post, but you don't always get what you
want. This is a good reminder that tests are not perfect, and that they're a
tool to alert you that something may have gone wrong, not proof that there's an
error. Test code can be fragile or have bugs, too. But without these tests,
we wouldn't have realized that there was a regression, no matter how small.

It's hard to get any complex software system behavior correct. So far, we've
found that these external tests give us a really nice forum for working out
these kinds of issues. They also give us something to share with our customers,
and a way to ask for advice from experts outside of the company.