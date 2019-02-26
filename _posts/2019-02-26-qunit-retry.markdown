---
layout: post
title: "qunit-retry"
comments: true
categories: [JavaScript]
---

Over the weekend I built a simple utility for QUnit, [qunit-retry](https://github.com/mrloop/QUnit-retry). [QUnit](https://qunitjs.com/) is a JavaScript Unit Testing framework which I use extensively daily not just for unit testing but acceptance and integration testing in [Ember.js](https://emberjs.com/). The majority of the acceptance tests are written against [mirage](http://www.ember-cli-mirage.com/), a client side server that mocks out the real API. For a few critical paths through the app, the project has acceptance tests against its real API and interacts with third party services such as payment processors using a test account.

A couple of these critical path tests were failing intermittently on the projects continuous integration service and it proved difficult to debug the test failure. The tests passed consistently when run locally on development computers. They passed the vast majority of the time and would very occasionally fail. The most pragmatic approach to resolve very occasionally failing test on the CI service which were maybe only fixable with considerable and repeated effort was to retry the test on failure to see if it passed. If it passed on retry consider the test successful, if it failed again consider it a failure. There's no built in `retry` test function for QUnit so I wrote a small npm package, creating a `retry` function which is a drop in replacement for QUnit [`test`](https://api.QUnitjs.com/QUnit/test) function, but will retry a test upon failure.

```js
retry('test critical path which occasionally fails on CI', function(assert) {
  let result = theTest();
  assert.equal(result, 42);
});
```

The tests now run green, no more intermittent test failures, and no more wondering if it's a real failure or not and spending time investigating, leaving more time to buiding the application.
