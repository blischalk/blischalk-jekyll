---
layout: post
title: "Twitterfeed bit.ly URL Links Broken and Link to Tinyurl"
description: "After setting up a Twitterfeed account to populate my social networking profiles with data from my blog’s Atom feed, I immediately noticed that the shortened bit.ly links that Twitterfeed was sending to my social media sites were broken. Even though I had selected bit.ly as the service I would like to use to shorten my url’s, the links would show up on Twitter, Facebook, and Linkedin as Tinyurl links"
redirect_from:
  - /posts/8-twitterfeed-bit-ly-url-links-broken-and-link-to-tinyurl
tags: [atom, twitterfeed]
---

After setting up a Twitterfeed account to populate my social networking profiles with data from my blog's Atom feed, I immediately noticed that the shortened bit.ly links that Twitterfeed was sending to my social media sites were broken.  Even though I had selected bit.ly as the service I would like to use to shorten my url's, the links would show up on Twitter, Facebook, and Linkedin as Tinyurl links. ( Tinyurl is another service which I don't currently use and hadn't selected ).

After searching around the internet I came across this post:
[https://getsatisfaction.com/twitterfeed/topics/why_does_my_twitterfeed_post_with_a_tinyurl_link_when_bit_ly_is_selected](https://getsatisfaction.com/twitterfeed/topics/why_does_my_twitterfeed_post_with_a_tinyurl_link_when_bit_ly_is_selected)

The response by Rex Dixon seemed to solve my problem. "A common reason for this is that the link elements don't contain the whole URL, i.e. they don't contain the full "http://domain.com/etc/".

After looking at my posts_feed.atom.builder template file I noticed that I had:

{% highlight ruby %}
feed.entry(post, :url => "/posts/#{post.to_param}") do |entry|
{% endhighlight %}

Sure enough, I was telling my atom feed that the post path was root relative instead of absolute.  I changed the previous line to:

{% highlight ruby %}
feed.entry(post, :url => "http://brettlischalk.com/posts/#{post.to_param}") do |entry|
{% endhighlight %}

and my links seemed to start working properly.
