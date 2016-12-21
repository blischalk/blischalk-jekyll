---
layout: post
title: "Associations in ActiveRecord \"not\""
description: "While working with Rails 4 today I was attempting to use the new .not method of ActiveRecord. This method is similar to the jQuery .not method as it allows you to filter your result set where a particular attribute of your result does NOT match what you pass in as .not’s argument. The syntax is:"
redirect_from:
  - /posts/13-associations-in-activerecord-not
tags: [activerecord, rails4]
---

While working with Rails 4 today I was attempting to use the new .not method of ActiveRecord.  This method is similar to the jQuery .not method as it allows you to filter your result set where a particular attribute of your result does NOT match what you pass in as .not's argument.  The syntax is:

{% highlight ruby %}
Post.where.not(author: author)
{% endhighlight %}

This will also insure that when author is nil the proper sql syntax of IS NOT NULL is used.

I couldn't find much out on the internet about this method aside from what I had seen on [CodeSchool's Rails 4: Zombie Outlaws](http://www.codeschool.com/courses/rails-4-zombie-outlaws) and I needed to perform a "not" based on an association.  The code school example only covered the simple scenario of a not based on a field on the model you are performing the not on.  Here is what worked for associated models:

@@@ruby
{% highlight ruby %}
scope :purchasable, ->{
    includes(:table2)
    .includes(:table3)
    .where.not(table3: {table2_id: nil})
    #.where('table3.table2_id IS NOT NULL')
}
{% endhighlight %}

Long story short, you need to use nested hashes to utilize associations in your not method calls.
