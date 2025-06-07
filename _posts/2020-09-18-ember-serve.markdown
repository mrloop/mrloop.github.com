---
layout: post
title: "Improving Ember.js serve and testing performance"
comments: true
categories: [JavaScript, Ember, Typescript]
excerpt: "I use the cli command ember serve or the abbreviated form ember s everyday, to build and serve locally ember apps for development. I noticed that ember"
---

I use the cli command `ember serve` or the abbreviated form `ember s` everyday, to build and serve locally ember apps for development. I noticed that `ember s --path dist` and `ember t --path dist` were taking a long time to start. The `--path` flag lets you reuse an existing build at the given path.

```sh
➜  ~ ember help | grep -- "--path\|^\w" | grep -B1 -- "--path"
ember serve <options...>
  --path (Path) Reuse an existing build at given path.
ember test <options...>
  --path (Path) Reuse an existing build at given path.
```

## Slow Continuous Integration

This long start time also significantly increased the total run time of the continuous integration, CI, test run of the project I was working on. The CI build was split into a compile stage and a test stage. The compile stage fetches dependencies and compiles the app using `ember b` and prepares the server side API for our full stack smoke tests. The app uses ember-cli-typescript, and the build can take around 2 minutes compiling typescript on the CI. The test stage consists of 7 concurrent jobs which all use the compiled assets from the compile stage. By having a single compile stage and 7 test stages the start time to end time will be quicker. The total run time will be slightly longer due to multiple 7 extra test environment boot steps.

<figure class="flex justify-center m-8">
  {% include ci-before.svg %}
  <figcaption class="hidden">Continuous Integration setup</figcaption>
</figure>

However the total run time for the CI test run increased more than could be accounted for by the extra boot steps, that is the total time for all the jobs to complete took around 14 minutes more than expected. Due to using concurrent jobs the time from starting the build and the build finishing reduces, there was a decrease in time to feedback but not what was expected and the total run time increased more than expected.

## What happens when you run `ember t`?

`ember` is the [command line interface for Ember.js](https://cli.ember.com.) `whereis ember` will tell you where the executable is, in my case `~/.yarn/bin/ember`. Opening `~/yarn/bin/ember` the first line is `#!/usr/bin/env node` telling us its a node app, which creates a ember-cli [`CLI`](https://github.com/ember-cli/ember-cli/blob/f1d58c22bb70b2c83626abff1e361e860661580d/lib/cli/index.js#L123) and [`run`s](https://github.com/ember-cli/ember-cli/blob/f1d58c22bb70b2c83626abff1e361e860661580d/lib/cli/index.js#L145) it. Parsing the command line args and creating a Command object with in turn creates the appropriate Task object and runs it. In this case it's the [TestTask](https://github.com/ember-cli/ember-cli/blob/069dbf3353daad643fc6bd46840b7d1435e41c2f/lib/tasks/test.js), which when run invokes [testem](https://github.com/testem/testem), the JS test runner used by Ember.js. The TestTask also allows ember addons to [inject middleware](https://github.com/ember-cli/ember-cli/blob/069dbf3353daad643fc6bd46840b7d1435e41c2f/lib/tasks/test.js#L30) into testem. This is very similar to the ServerTask, run when using `ember s`, with also allows ember addons to [inject middleware](https://github.com/ember-cli/ember-cli/blob/836e89b4f26577cc9e492973048b9293b375c914/lib/tasks/server/express-server.js#L100) into the development server.

## Identifying slow build stage

There's some useful documentation describing how to track down [build performance issues here](https://github.com/ember-cli/ember-cli/blob/c9320aeb9fb521887bd8fec8f722bef0608c9fe5/docs/perf-guide.md#debug-logging) and in the [ember-cli docs](https://cli.emberjs.com/release/advanced-use/debugging/). Using one of the techniques described, I add a `DEBUG` environment variable before `ember s`


```sh
$ DEBUG==ember-cli:* ember s --path dist
  ember-cli:test-server isForTests: false +10s
  ember-cli:broccoli-watcher serving: /admin/index.html +37s
  ember-cli:broccoli-watcher serving: (prefix stripped) /index.html, was: /admin/index.html +1ms
```

Why does it take 10s for isForTests, and what is broccoli-watcher doing, I thought we were serving precompiled assets? To dig down a bit more, and produce a lot more debug info ran `DEBUG=*:* ember s --path dist`

```sh
$ DEBUG=*:* ember s --path dist
...
– Serving on http://localhost:4200/admin/
...
ember-cli-typescript:typecheck-worker Typecheck complete (0 diagnostics) +49s
...
```

It's not `ember-cli:broccoli-watcher` but `ember-cli-typescript:typecheck-worker` taking up all the time. The real culprit is ember-cli-typescript middleware. Type checking is being performed even when the code is already transpiled, the case when using the `--path` flag.

## Improving performance

The code change to improve performance was [small](https://github.com/typed-ember/ember-cli-typescript/pull/1148/commits/df022189ffd144fe70ba7bb5551cc3fdde565f0c#diff-287bcabd2443a8d80ebe0fb212241315). When adding server middleware or testem middleware check the options passed to ember-cli and if they contain the path flag then don't run typechecking. A small [`ember-cli` update](https://github.com/ember-cli/ember-cli/pull/9205) was also required. ember-cli would pass all the CLI flags to the server middleware, but not to the testem middleware. Without the ember-cli update, ember-cli-typechecking would not of been able to perform the check when testem middleware was added. Interesting to note that for each PR I spent a lot more time figuring out how to effectively test the changes than implement them.

<figure class="flex justify-center m-8">
  {% include ci-after.svg %}
  <figcaption class="hidden">Decreased CI test time after performance improvement</figcaption>
</figure>

These updates significantly sped up the CI build times, and more noticeably, improved my work flow by drastically reducing execution time of the ember commands when using the `--path` flag.
