---
layout: post
title: "Data Munging in Ruby on Rails"
description: "It has been awhile since my last blog post and quite a lot has happened. Since my last blog post I have completed a large Drupal 5 to Drupal 7 migration (The d5 to d7 migration has actually been an ongoing endeavor that was finally completed after a years worth of development), implemented authorization via cancan on this blog, and implemented nested comments via polymorphic associations and the ancestry gem on this blog. I should have quite a lot to talk about now!"
redirect_from:
  - /posts/9-data-munging-in-ruby-on-rails
tags: [data, munging, migrations, ruby, rails, rake]
---

It has been awhile since my last blog post and quite a lot has happened.  Since my last blog post I have completed a large Drupal 5 to Drupal 7 migration (The d5 to d7 migration has actually been an ongoing endeavor that was finally completed after a years worth of development), implemented authorization via cancan on this blog, and implemented nested comments via polymorphic associations and the ancestry gem on this blog.  I should have quite a lot to talk about now!

Today's blog post is actually about none of those topics.  During the process of deploying my nested commenting feature to this blog today I encountered a strange error which prevented my deployment from completing successfully.  A sort of chicken or egg scenario if you will that was quite perplexing.  While capistrano was trying to run my migrations on the production server I received the following error:

{% highlight bash %}
undefined method `user_name' for #<User:0xbe8bd08>
{% endhighlight %}

What was maddening is that my migrations had worked just fine in my development environment but for some reason wouldn't complete properly on production.  What turned out to be the cause of this???  Attempting to perform data munging within migrations.  In a nutshell, here is how everything went down:

* Added comment table
* Added authorization table
* Since I now implemented authorization via roles I needed to assign roles to existing users.  I decided to perform this data munging via a migration.
* Added user names to users so comments would display who they were written by.

It all seems pretty straight forward right?  Well after some help from "alindeman" on IRC I was informed that it is better to perform data munging via rake tasks instead of migrations because of the exact scenario that I was encountering.  But how could the data munging in the migrations be causing the problem?  This is how:

1. When I was implementing the steps outlined above the first time in development everything worked fine because the validation on the user_name field was not implemented in my model until after the data munging migration had occurred.
2. When I was trying to deploy to production and these steps were being executed there, the data munging migration would fail because the migration that adds the user name column to the user table runs after the data munging role migration which calls user.save.

Confused?  Here is some code that might make more sense:

{% highlight ruby %}
# Data munging migration that runs first
class CreateAuthenticatedRole < ActiveRecord::Migration
  def up
    auth = Role.find_or_create_by_name('Authenticated')
    users = User.all
    users.each do |u| 
      u.roles << auth
      u.save!
    end 
  end 

  def down
    users = User.all
    role = Role.find_by_name('Authenticated')
    users.each do |u| 
      u.roles.delete(role)
      u.save!
    end 
    Role.delete(role)
  end 
end
{% end highlight %}

{% highlight ruby %}
# Migration that adds username and runs after the previous data munging migration
class AddNameToUsers < ActiveRecord::Migration
  def up
    add_column :users, :user_name, :text
    User.update_all(user_name: 'anonymous')
    user = User.find(9999)
    user.user_name = 'foo'
    user.save
  end 
  def down
    remove_column :users, :user_name
  end 
end
{% end highlight %}

And then the last piece of the puzzle, the user model that went with the migration

{% highlight ruby %}
class User < ActiveRecord::Base
  validates :user_name, uniqueness: true
  validates :user_name, presence: true
 ...more code 
end
{% endhighlight %}

The first time the migrations executed, no problem because I hadn't implemented validation on the username field of the user model yet.  Second time through, failure because code was conflicting with the database schema.

Moral of this story is that I will now perform any data munging via rake tasks and leave my migrations for schema updates only.
