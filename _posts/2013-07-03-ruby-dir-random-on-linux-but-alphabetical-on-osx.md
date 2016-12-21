---
layout: post
title: "Ruby Dir \"Random\" on Linux but Alphabetical on OSX"
description: "Recently after a co-worker pushed code to our continuous integration server Jenkins was saying that the build was broken and that a constant was being called that wasn’t initialized. What was strange is that the code my co-worker pushed ran just fine when I ran it on my work Mac Laptop as well as some of my other co-workers Mac Laptops. This has to be some sort of weird Mac / Linux difference."
redirect_from:
  - /posts/14-ruby-dir-random-on-linux-but-alphabetical-on-osx
tags: [jenkins, linux, mac, rspec, ruby, rails]
---

Recently after a co-worker pushed code to our continuous integration server Jenkins was saying that the build was broken and that a constant was being called that wasn't initialized.  What was strange is that the code my co-worker pushed ran just fine when I ran it on my work Mac Laptop as well as some of my other co-workers Mac Laptops.  This has to be some sort of weird Mac / Linux difference.

I ran the code on a Ubuntu system (our CI server was CentOS) and sure enough it blew up with the same undefined constant error.  This seemed to prove that this had something to do with the Mac / Linux difference between our development machines and our CI server.

In our spec/spec_helper.rb file is the line

{% highlight ruby %}
Dir[Rails.root.join("spec/support/**/*.rb")].each {|f| require f}
{% endhighlight %}

This is the line that includes all of the support files for the Rspec specs.  I ran a pry on ~Dir[Rails.root.join("spec/support/**/*.rb")]~ on the linux box and then again on my mac and sure enough on Mac:

{% highlight bash %}
support/models_helper/cas/server_helpers.rb
spec/support/models_helper.rb
{% endhighlight %}

On Linux:
{% highlight bash %}
spec/support/models_helper.rb
support/models_helper/cas/server_helpers.rb
{% endhighlight %}

The order in which Ruby's "Dir" command was returning its results was alphabetical on Mac and what seemed to be random on Linux.  After doing a little research I came upon this [StackOverflow thread](http://stackoverflow.com/questions/5529895/why-does-ruby-seem-to-access-files-in-a-directory-randomly).

It seems that this is not an issue with Ruby but just a difference in operating systems.  The line in the spec helper file that says, **"Requires supporting ruby files with custom matchers and macros, etc, in spec/support/ and its subdirectories."** appears to be slightly inaccurate.  To get around this issue we just ended up calling require at the top of the support file where the constant was referenced and that ensured that the constant was loaded before the file referenced the constant.
