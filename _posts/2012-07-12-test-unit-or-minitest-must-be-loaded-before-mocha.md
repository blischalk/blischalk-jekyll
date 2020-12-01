---
layout: post
title: "Test::Unit or MiniTest must be loaded *before* Mocha"
description: "I recently ran bundle update on an older rails application that I had been developing. I added a few specs and when I went to run them I was greeted with the following error:"
redirect_from:
  - /posts/10-test-unit-or-minitest-must-be-loaded-before-mocha
tags: [gem, mocha, rails, rspec, ruby]
---

I recently ran bundle update on an older rails application that I had been developing. I added a few specs and when I went to run them I was greeted with the following error:

{% highlight ruby %}
Test::Unit or MiniTest must be loaded *before* Mocha (use MOCHA_OPTIONS=skip_integration if you know what you are doing).
{% endhighlight %}

I was scratching my head for a little while wondering if I had messed something up in my specs until it dawned on me that I had just previously ran bundle update on my app which had upgraded a few of my gems, in particular Mocha.  I went into my rvm gem directory for mocha and did a grep on the exception.  I found it within: mocha-0.12.0/lib/mocha/integration.rb.  When I did a diff between the same file in the mocha-0.11.4 version I saw that low an behold the exception had been added to that integration.rb in the 0.12.0 version of the mocha gem.

Now that I had tracked down the error, what do I do to get rid of it so I can run my specs!

I found the mocha docs [here](http://gofreerange.com/mocha/docs/)

After reading through them I noticed a note about bundler:

"If you’re using Bundler, ensure the correct load order by not auto-requiring Mocha in the Gemfile and then later load it once you know the test library has been loaded…"

{% highlight ruby %}
# Gemfile
gem "mocha", :require => false
{% endhighlight %}

I had just used bundler and I didn't recall having the :require => false argument in my gem file.  I added it to my gem file and presto!  My specs began working again.
