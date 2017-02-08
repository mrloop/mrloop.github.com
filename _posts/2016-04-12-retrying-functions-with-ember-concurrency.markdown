---
layout: post
title: "Retrying functions with Ember Concurrency"
date: 2016-04-12 21:07
comments: true
categories: [Javascript, Ember]
---

I recently wrote about using [ember-retry](https://www.npmjs.com/package/ember-retry) in a previous [blog post]({% post_url 2016-03-26-retrying-functions-in-ember %}) to retry a function in ember with exponential backoff. The next day I watched the [Ember.js NYC](http://www.youtube.com/watch?v=uVr5HWzecKI&t=1h08m05s) video about [ember-concurrency](http://ember-concurrency.com) specifically [Loop until operation succeeds, with exponential backoff](http://www.youtube.com/watch?v=uVr5HWzecKI&t=1h51m26s)

So how do you implement exponential backoff with ember-concurrency?

```javascript
import Ember from 'ember';
import { task, timeout } from 'ember-concurrency';

const MAX_RETRIES = 5;
const WS_URL = 'ws://ws.qustionr.com'

export default Ember.Component.extend({

  init: function(){
    this._super(...arguments);
    this.get('setupWs').perform();
  },

  setupWs: task(function * () {
    let delay = 500;
    for(var i=0; i < MAX_RETRIES; i++) {
      yield timeout(delay);
      delay = Math.pow(2, i) * delay;
      try{
        let webSockect = yield this.getWs(WS_URL);
        //do something with webSocket
        return webSocket;
      } catch(e){ }
    }
  }),

  getWs: function(url) {
    return new Ember.RSVP.Promise((resolve, reject)=>{
      try {
        let ws = new WebSocket(url)
        ws.onopen = ()=> resolve(ws);
        ws.onerror = (error)=> reject(error);
        ws.onclose = ()=> this.get('setupWs').perform();
      } catch(e) {
        reject(e);
      }
    });
  }

});
```
