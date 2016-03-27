---
layout: post
title: "Ember Data and the Meetup API"
date: 2013-04-07 15:45
comments: true
categories: [Javascript, Ember, Ember Data]
---

# Warning: Uses Deprecated Ember Data 0.12

I'm a member of the highland web group and they were looking for somebody to
create a website for the group. Alisdair and myself were interested in doing
so. The group's organized via [meetup.com](http://www.meetup.com/Highland-Web-Group/) and they have
a fairly comprehensive [API](http://www.meetup.com/meetup_api/) for accessing
their data. The Meetup data should always be up to date so instead of burdening
somebody with the task of updating meetup with current information and our own
website with an alternative CMS the best option was to populate
the [highlandweb group](http://highlandwebgroup.github.com) with meet up data dynamically.
To do this I decided to use [Ember](http://emberjs.com) and [Ember Data](http://emberjs.com/guides/models/).

Ember is a MVC javascript framework for creating web applications, Ember Data is a data persistence framework for Ember.
Ember Data provides a central Data Store and Adapters to talk to different data sources.
Ember and Ember Data believe in convention over configuration, the framework will do a lot of work for you if you follow the conventions.
This can make things a little tricky if you step away from those conventions.

The meetup API is not 100% consistent in how it defines object ids e.g. [RSVPs](#rsvps)
have their ids declared as 'rsvp_id' whilst most other models simply use 'id'.
The RSVPs also use a '-1' id as a flag indicating the RSVP is a recurring rsvp
from the events organizer. Also nested objects aren't consistent or always
complete e.g. [Groups](#groups).

The inconsistent JSON structure returned by the Meetup API make it difficult to write an ember data adapter for it.
Luckily ember data have recently addressed this issue with the [Basic Adapter](http://emberjs.com/blog/2013/03/22/stabilizing-ember-data.html)
letting you write a sync object for specific models in your API so you can work around the inconsistencies.

Sync Object
-----------

I define a function *meetupSync* which returns a generic sync object.

```js
//https://github.com/HighlandWebGroup/highlandwebgroup.github.com/blob/master/app/lib/MeetupAdapter.js
App.meetupSync = function(type){
  sync = {
    munge: function(json){json},
    findHasManyQueryOptions: {},
    base_url:"https://api.meetup.com/2",
    find: function(id, process){
      q = {};
      q[this.type+"_id"] = id;
      _munge = this.munge;
      this._query(q, function(result){
        process(result.results.length > 0 ? result.results[0]  : [])
          .munge(_munge)
          .load();
      });
    },
    query: function(query, process){
      _munge = this.munge;
      this._query(query, function(result){
        process(result.results || [])
          .munge(_munge)
          .load();
      })
    },
    findHasMany: function(record, options, processor){
      relationship_model = record.constructor.typeForRelationship(options.relationship)
      q = {};
      q[record.constructor.sync.type+"_id"] = record.id;
      relationship_model.sync.query(Ember.$.extend({},q, this.findHasManyQueryOptions), processor)
    },
    _query: _.rateLimit(function(query, processResult){
      Ember.assert("Must specify type", this.type);
      App.ajax(this.base_url+"/"+this.type+"s", "GET", {data: query}).then(processResult);
    }, 100),
  }
  sync_copy = Ember.$.extend(true, {}, sync)
  sync_copy.type = type;
  return sync_copy;
}
```

This is a generalized sync object for the meetup API that I can customise for each model
as needed. I define 3 functions for finding records from the API.

  * *find* for finding records by Id
  * *query* for finding records by other parameters
  * *findHasMany* for finding related records.

In my implementation *find* and *findHasMany* both use *query* to find records.
The *query* function takes 2 arguments, 'query' e.g.

``` js
{ status: 'upcoming' }
```
and a a function 'process' with we use to load the JSON from our request into the Data Store after we have done our own
processing with our *munge* function. The query function uses our private
function *_query* which is [rate limited](https://gist.github.com/mattheworiordan/1084831) to stop meetup API restricting access
for too frequent API calls.

Models
------

### Event Models

```js
//https://github.com/HighlandWebGroup/highlandwebgroup.github.com/blob/master/app/lib/models/event.js
var attr = DS.attr;

App.Event = DS.Model.extend({

  rsvps: DS.hasMany('App.Rsvp'),
  photos: DS.hasMany('App.Photo'),

  group: DS.belongsTo('App.Group'),

  attending_rsvps: function(){
    return Ember.ArrayController.create({
      content: this.get('rsvps').filterProperty('response','yes')
    });
  }.property('rsvps.@each'),

  event_url: attr('string'),
  status: attr('string'),
  name: attr('string'),
  time: attr('date'),
  description: attr('string'),
  how_to_find_us: attr('string'),
  venue: attr('object')
})
```

The Event model is defined then its sync object is initialised.

``` js
App.Event.sync = App.meetupSync('event');
```

The meetup API return an events data including the events group as a nested object but without
all the information we want to load a App.Group object into the data store.
<a id='groups'></a>

*events.json http://www.meetup.com/meetup_api/console/?path=/2/events*

```json
{
  "results": [
  {
    "status": "upcoming",
      "visibility": "public",
      "maybe_rsvp_count": 0,
      "venue": {
        "id": 6200762,
        "lon": -4.2325,
        "repinned": false,
        "name": "Eden Court",
        "address_1": "Bishops Road",
        "lat": 57.472176,
        "country": "gb",
        "city": "Inverness"
      },
      "id": "ghnjqyrgbvb",
      "utc_offset": 3600000,
      "time": 1366135200000,
      "waitlist_count": 0,
      "updated": 1331810527000,
      "yes_rsvp_count": 2,
      "created": 1296155663000,
      "event_url": "http://www.meetup.com/Highland-Web-Group/events/111106332/",
      "description": "<p>An informal and friendly setting for chat about all types of web-related stuff. All welcome.</p>",
      "how_to_find_us": "Meet in the Tulloch Room. Ask at reception if you're unsure how to find it.",
      "name": "The Highland Web Group Monthly Meetup",
      "headcount": 0,
      "group": {
        "id": 1744559,
        "group_lat": 57.47999954223633,
        "name": "Highland Web Group",
        "group_lon": -4.230000019073486,
        "join_mode": "open",
        "urlname": "Highland-Web-Group",
        "who": "Members"
      }
  }]
}
```
A custom *munge* function is defined for handling results returned from the
API.

```js
//event.js https://github.com/HighlandWebGroup/highlandwebgroup.github.com/blob/master/app/lib/models/event.js
App.Event.sync.munge = function(json){
  if(json.group && json.group.id){
    json.group_id = json.group.id;
    json.group = null;
  }
}
```

The group id is extracted from the nested object and set the group_id key on the
top level event object so ember data properly handles the belongsTo
relationship. The last thing to do is set the JSON group object to null so
ember data doesn't try and materialise a App.Group from this record.

### <a id="rsvps"></a>Rsvp Model

The data returned for an rsvp is structure slightly differently to that of an Event.

*rsvps.json http://www.meetup.com/meetup_api/console/?path=/2/rsvp*

```json
{
  "results": [
  {
    "response": "yes",
      "member": {
        "name": "Blair Millen",
        "member_id": 8357569
      },
      "member_photo": {
        "photo_link": "http://photos2.meetupstatic.com/photos/member/3/b/e/f/member_5175343.jpeg",
        "highres_link": "http://photos2.meetupstatic.com/photos/member/3/b/e/f/highres_5175343.jpeg",
        "thumb_link": "http://photos2.meetupstatic.com/photos/member/3/b/e/f/thumb_5175343.jpeg",
        "photo_id": 5175343
      },
      "created": 1364297767000,
      "event": {
        "id": "ghnjqyrgbvb",
        "time": 1366135200000,
        "event_url": "http://www.meetup.com/Highland-Web-Group/events/111106332/",
        "name": "The Highland Web Group Monthly Meetup"
      },
      "mtime": 1364297767000,
      "guests": 0,
      "rsvp_id": 741139992,
      "venue": {
        "id": 6200762,
        "lon": -4.2325,
        "repinned": false,
        "name": "Eden Court",
        "address_1": "Bishops Road",
        "lat": 57.472176,
        "country": "gb",
        "city": "Inverness"
      },
      "group": {
        "id": 1744559,
        "group_lat": 57.47999954223633,
        "group_lon": -4.230000019073486,
        "join_mode": "open",
        "urlname": "Highland-Web-Group"
      }
  }]
}
```

The Rsvp model looks like this.

```js
//rsvp.js https://github.com/HighlandWebGroup/highlandwebgroup.github.com/blob/master/app/lib/models/rsvp.js
var attr = DS.attr;

App.Rsvp = DS.Model.extend({

  event: DS.belongsTo('App.Event'),
  memberLink: function(){
    return "http://www.meetup.com/members/"+this.get('member.member_id');
  }.property('member'),
  comments: attr('string'),
  created: attr('date'),
  member: attr('object'),
  member_photo: attr('object'),
  response: attr('string'),

})

App.Rsvp.sync = App.meetupSync('rsvp');
```

The Rsvp models munge function does a little more work.

```js
//rsvp.js https://github.com/HighlandWebGroup/highlandwebgroup.github.com/blob/master/app/lib/models/rsvp.js
App.Rsvp.sync.munge = function(json){
  if(json.rsvp_id && json.rsvp_id == -1){
    //ember-data wont accept -1 id
    json.id = "RecurringOrganizersRsvpId";
  }else{
    json.id = json.rsvp_id;
  }
  if(json.event){
    json.event_id = json.event.id;
    json.event = null;
  }

  return json;
}
```

The rsvp data returned by the API uses 'rsvp_id' instead of 'id' for the object id and it also sets the rsvp_id to -1 if it a repeating event that hasn't been rsvp'd by anybody but the host. ember-data doesn't handle -1 id so we set this to something else or if an ordinary id is returned we change the key from rsvp_id to id.

As with the Event result data the Rsvp result data contains a nested object. In this case it's the data for the event the rsvp belongs to. We extract the events id and set the JSON Event object to null.

Security
--------

[JSONP](http://www.json-p.org) or [CORS](http://www.w3.org/wiki/CORS) can be used to make requests from different domains to the Meetup
API. The Meetup CORS requires OAuth authentication but we don't want to require
website visitors signing in to view publicly available information. JSONP on
the other hand requires an API key and because we're building a javascript app
this API key will be viewable by anybody if they look at the javascript files.
Ideally Meetup API key authentication could be tied to a specific domain that
can only be specified via their web dashboard so me as the owner of the key
could specify that it should only work for [highlandwebgroup.github.com](http://highlandwebgroup.github.com)


