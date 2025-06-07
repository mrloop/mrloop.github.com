---
layout: post
title: "Know the competition"
comments: true
categories: [JavaScript, GlimmerJS]
excerpt: "I do quite a bit of cycling, and have been thinking about racing in some competitive events again. A lot of the races are organized by British"
---

I do quite a bit of cycling, and have been thinking about racing in some competitive events again. A lot of the races are organized by British Cycling and for many of the competitive events lists of entrants are published. You can view entrants and their national and regional rankings based on points they've won for good placings in previous races.
But there is no collated view of entrants and their points. What I want is a list of entrants ordered by there national and regional rankings to quickly see who to watch in a race.

My original notion was to build a simple API endpoint using [serverless](https://serverless.com/), the single endpoint, `GET /competition/{id}` would take the id of a race, scrape the relevant British Cycling webpages, cache them where possible and return a list of entrants ordered by ranking.
A quick prototype confirmed this was possible but had a pretty dismal user experience. You had to go the britishcycling.org.uk find the race and then copy and paste the id into a web form and then get the returned results.

Ideally I want to list and search races and then select the race I'm interested in from the same user interface. There were two ways I thought would work well.

* A simple command line interface.
* A web extension which inserts a button into britishcycling.org.uk against each event listing which when pressed lists the entrants. Also will have a larger audience.

### [race-lib](https://github.com/mrloop/race-lib)

The first step was to extract scraping logic from prototype it should be

* isomorphic, that is code will run in browser and in node
* injectable dependencies for fetching web pages and querying them.
* packaged for easy reuse
* include tests

As part of [Ember.js](https://emberjs.com/blog/2017/07/06/ember-2-14-released.html) I've been using [Rollup](https://rollupjs.org/), but haven't configured and used outwith of Ember.js. Rollup will give me a ES6 standardized module I can use in other projects. Re usability of lib is further increased by using injectable dependencies for fetching URLs and for querying html. For example in node [Cheerio](https://www.npmjs.com/package/cheerio) is used for parsing and querying html. [node-fetch](https://www.npmjs.com/package/node-fetch) a small library which brings the `window.fetch` API to node is used.


#### Inject

```js
import { Event } from 'race-lib';
import cheerio from 'cheerio';
import fetch from 'node-fetch';

Event.inject('fetch', fetch);
Event.inject('cheerio', cheerio);

Event.upcomming().then((events) => {
  events.forEach( evt => console.log(evt.name));
});
```

In the browser `window.fetch` is used and jQuery used for query the html.

```js
$.load = function(htmlString: string) {
  const o = $($.parseHTML(htmlString));
  return function (selector) {
    return o.find(selector);
  }
}

Event.inject('fetch', window.fetch);
Event.inject('cheerio', $);

Event.upcomming().then((events) => {
  events.forEach( evt => console.log(evt.name));
});
```

#### Throttle

While developing and testing `race-lib` it became apparent that requests to britishcycling.org.uk is rate limited and a 429 is returned if you exceed 1 request per 4 seconds over about a minute. The throttling of requests in the lib is dumb at the moment, requests are serialized and a 4 second wait between them added. The library allows you to register for notifications about progress so you can notify the user about the progress of fetching rankings.

```js
race.addEventListener('entrantLoaded', (data) => {
  let status = `Entrants loading ${data.detail.loaded}/${data.detail.total}`;
  displayEntrants(data.detail.users, status);
});
```

### [race-cli](https://www.npmjs.com/package/race-cli)

A simple command interface for retrieving data about entrants to British Cycling races. It is a wrapper around [race-lib](https://github.com/mrloop/race-lib) which provides a command line prompt with which to search races and list entrants.

To install and run use the following commands:

```sh
npm install -g race-cli
race
```

It allows you to search for events and browse them. Upon selecting a race it displays a table of entrants.

<asciinema-player font-size="15" loop="true" autoplay="true" src="/video/race-cli.json"></asciinema-player>
<script src="/js/asciinema-player.js"></script>

### [Know the competition](https://addons.mozilla.org/en-US/firefox/addon/know-the-competition/)

[Source code here](https://github.com/mrloop/race-ext)

Next was to write a web extension running [race-lib](https://github.com/mrloop/race-lib) and doing some web scraping in browser. I've been using [EmberJS](https://emberjs.com/) a lot recently. For the web extension a lot of the functionality in [EmberJS](https://emberjs.com/) is not needed, and would add unneeded bloat. [GlimmerJS](https://glimmerjs.com/) has recently been released and I wanted to try it out for something other than a hello app and this seemed a good excuse.

<video controls poster="/video/know-the-competition.png" src="/video/know-the-competition.m4v"></video>

Each `Entrants` button is a [GlimmerJS](https://glimmerjs.com/) component inserted into the existing web pages DOM, the modal window is another [GlimmerJS](https://glimmerjs.com/) component which is triggered to via `window.postMessage`. [GlimmerJS](https://glimmerjs.com/) handles the presentation layer while [race-lib](https://github.com/mrloop/race-lib) fetches and queries the race data. [race-lib](https://github.com/mrloop/race-lib) has `window.fetch` injected to handle fetching other web pages for scraping while jQuery is injected to handle the html querying of these pages to extract the relevant data.

### Future improvements

To reduce the delay caused by throttling, caching British Cycling requests would be desirable. This could be done at the [race-lib](https://github.com/mrloop/race-lib) level, the [race-lib](https://github.com/mrloop/race-lib) could take an injectable `cache-store` that could be changed depending on the environment. Or could move the web scraping to a web service which handles the scraping / caching server side, the cache would be shared between users, more likely to hit the cache and it would reduce the amount of requests to British Cycling.

To increase the possible audience for the web extension could release a web extension for Edge and Chrome. With a unified [WebExtension](https://developer.mozilla.org/en-US/Add-ons/WebExtensions) API this should be simple right?
