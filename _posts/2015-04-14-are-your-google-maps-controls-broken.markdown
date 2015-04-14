---
layout:     post
title:      "Are your Google Maps controls broken?"
date:       2015-04-14 7:05:00
categories: google-maps
---
While experimenting with Jekyll for the first time, I noticed that when I attempted to bring up a Google Map in an example page it had broken controls. If you've worked with Google Maps in a site using Foundation or Bootstrap, you've probably seen something like this at some point:

![Screenshot of broken map controls]({{site.baseurl}}/img/map-controls-broken.png)

This happens because responsive frameworks like to set `max-width: 100%` on images. In this case to get the controls displaying properly again, you just need to override the `max-width` property on images inside your map containing element. Something like this:

{% highlight css %}
.map-container img {
  /* Override the global "max-width: 100%" setting on img */
  max-width: none;
}
{% endhighlight %}

