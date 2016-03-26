---
layout: post
title: "Retrying functions in Ember"
date: 2016-03-26 15:40
comments: true
categories: [Javascript, Ember]
---

For a recent project [questionr.com](https://questionr.com) I needed to handle the deployment of a WebSocket server, the server would go down for a couple of seconds during the deploy and the client would lose it's connection. To handle the reconnection I wrote a simple ember addon, [ember-retry](https://www.npmjs.com/package/ember-retry) which is used to retry the WebSocket server with an exponential backoff.

{% highlight javascript %}
import Ember from 'ember';
import retry from 'ember-retry/retry';

export default Ember.Component.extend({

  init: function(){
    this._super(...arguments);
    this.setupWs();
  },

  setupWs: function(){
    retry((resolve, reject)=>{
      let ws = new WebSocket('ws://ws.qustionr.com');
      ws.onopen = ()=> resolve(ws);
      ws.onerror = (error)=> reject(error);
      ws.onclose = ()=> this.setupWs();
    }).then((webSocket)=>{
      //do something with webSocket
    });
  }

});
{% endhighlight %}

Where a library already uses promises, retry can be invoked without passing a function with resolve and reject arguments and instead returning the promise e.g.

```javascript
import Ember from 'ember';
import retry from 'ember-retry/retry';
import ws from 'monocules.ws-client';

export default Ember.Component.extend({

  init: function(){
    this._super(...arguments);
    this.setupWs();
  },

  setupWs: function(){
    retry(()=>{
      return ws.connect('name', {url: 'ws://ws.questionr.com'});
    }).then((webSocket)=>{
      webSocket.onclose = ()=> this.setupWs();
      //do something with webSocket
    });
  }
});
```
