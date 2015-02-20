---
layout: post
title:  "ActiveModel::Serializers considered harmful"
date:   2015-02-19 21:52:25
---

[ActiveModel::Serializers](https://github.com/rails-api/active_model_serializers) (AMS) is a rails plugin that provides serialization of ActiveModel objects into JSON. Sounds pretty nifty and like it could come in handy if I wanted a lightweight API without a bunch of serialization wonkyness going on all over the place.

I've had the oportunity to do a bit of work using these, so naturally, I've formed some opinions. I'll start by saying, most of the time these seem to work out alright. However, I've encountered a few senarios (2 to be exact) where I just can not stand them, and I've wasted too much time trying to get them to do what I want, and can't.

### null attributes included by default

Say we have a sparsely populated table. There are 30 columns in this table, all of them nullable (expect id of course), and a typical row only contains values in 2 or 3 columns.
By default, AMS includes ALL attributes that you've defined as accessible in the json response. Even in they are null.

EVEN IF THEY ARE NULL! 

`null` literally represents the absence of data, so why are we including null attributes? If they're not in the response, they're null! I don't need you to tell me this, especially on _every_ response.

This means, if we connect an API to this table and use AMS for serialization, our API is going to be shitting out 28 null attributes in the json response and at this point, the number of bytes representing null shit is larger than the number of bytes representing actual data.
Let's see what this looks like visually.

{% highlight ruby %}
{"boring_object": {
  "id": "blah",
  "attr1": "wtf",
  "attr2": "lol",
  "attr3": null,
  "attr4": null,
  "attr5": null,
  "attr6": null,
  "attr7": null,
  "attr8": null,
  "attr9": null,
  "attr10": null,
  "attr11": null,
  ... yadayada ..
  "attr29": null
}}
{% endhighlight %}

Maybe I'm insane, but this seems fucked. Why would I want that instead of

{% highlight ruby %}
{"boring_object": {
  "id": "blah",
  "attr1": "wtf",
  "attr2": "lol"
}}
{% endhighlight %}


This just does not make sense to me, and moreso that it's the default. We're wasting bandwidth, and potentially a shitload.

I have a hunch this may be intertwined with model validations somehow, but I haven't dug into it enough closely. And, unfortunately I wasn't able to find a good solution for this.


### Attribute values unintentially change

This _is_ as fucked as it sounds.

I was working on putting together a new JSON API endpoint and figured I'd give AMS a run since it's used in other places in this particular API.

Turns out if you have a column with a type of `DATE` (e.g. "2015-01-01") and run your object through AMS, the json comes out with something that resembles "Thrs, Jan 1 2015".

What the actual fuck? I know dates and the internet have never gotten along, but FUCK. How is this okay? And why?

I explicitly set the value of the column to "2015-01-01". The value in the database is "2015-01-01". But for some reason, AMS thinks it's okay to change the date format now that I want to serialize my object. And on top of that, it's a shit "pretty, human-readable" format.
Chances are a machine is going to interact with this json response, not a human, so I really can't see why this is a thing.

I haven't had the pleasure of digging deep enough into the AMS source to check this out yet, but I intend to, and I'll probably post findings here.

Anyway, the solution I ended up using to de-fuck this and avoid terrible, terrible date conversions happening all over the place was to use `to_json`.


#### Summing up

It's not my intent to be overly negative here or to hate on the authors of AMS (In fact, thanks for writing it!). I just wanted to shared the two scenarios I've run into where I've found AMS to break down for me personally.

