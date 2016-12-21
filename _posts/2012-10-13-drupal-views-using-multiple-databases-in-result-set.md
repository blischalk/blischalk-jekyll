---
layout: post
title: "Drupal Views: Using Multiple Databases in Result Set"
description: "Recently while working at my day job the need arose to utilize multiple databases to deliver a result in Drupal’s Views module. Not only did we need to utilize multiple databases but the databases reside on two different servers. I know that Drupal has the ability to connect to multiple databases but I had never considered how Views would utilize both databases or if it was even possible."
redirect_from:
  - /posts/12-drupal-views-using-multiple-databases-in-result-set
tags: [drupal, multiple, databases, views]
---

Recently while working at my day job the need arose to utilize multiple databases to deliver a result in Drupal's Views module.  Not only did we need to utilize multiple databases but the databases reside on two different servers.  I know that Drupal has the ability to connect to multiple databases but I had never considered how Views would utilize both databases or if it was even possible.

After some initial research I found the following posts on Drupal.org:

* [http://drupal.org/node/576694](http://drupal.org/node/576694)
* [http://drupal.org/node/816456#comment-3042936](http://drupal.org/node/816456#comment-3042936)

Essentially what I gathered from my research was that Views simply can't utilize two databases to generate it's results.  This was not the answer I was hoping for. :(

Before abandoning Views on this project in favor of a custom solution I decided to see if I could exploit custom Views Handler's, Views hooks, and the fact that Drupal supports connecting to multiple databases.  My theory was this:

1. Within a custom module I could define some pseudo columns on the node table within hook_views_data().
2. I would then add these columns to the view's fields within the Views module GUI.  Since there wouldn't actually be any table column associated with these columns there just wouldn't be any data in the result set for each row.
3. In hook_views_pre_render() I could exploit Drupal's ability to connect to multiple databases.  Here I have access to the view with the results already queried so I can run an additional query on my second database.
4. After running my second query in hook_views_pre_render to get the additional data from the second database I will just slap it onto the views object in a custom property.  I make sure that the data set is keyed by nid so that I can match up the data from the view with the data from my secondary query.
5. I would then create a custom Views handler that would format the field columns I had added via the custom module and Views GUI.  I access the additional data I had attached to the views object in the custom property and merge it into the rows output.

Here is some code to help illustrate the procedure.

_Note:** The following is just pseudo code to help illustrate the technique I used.  It is not cut and paste ready.  However if you are still confused on how the following code works, please feel free to ask questions in the comments._

First, make sure you have your secondary database setup in settings.php

{% highlight php %}
$databases = array (
  'default' =>  
  array (
    'default' =>  
    array (
      'driver' => 'mysql',
      'database' => 'primary_db_name',
      'username' => 'primary_db_uname',
      'password' => 'primary_db_pass',
      'host' => 'localhost',
      'port' => '3306',
      'prefix' => '', 
    ),  
  ),  
  'name_to_refer_to_database' =>  
  array (
    'default' =>  
    array (
      'driver' => 'mysql',
      'database' => 'secondary_db_name',
      'username' => 'secondary_db_uname',
      'password' => 'secondary_db_pass',
      'host' => 'secondary_server_host',
      'port' => '3306',
      'prefix' => '', 
    ),  
  ),  
);
{% endhighlight %}

In a custom module, implement the required hooks:

{% highlight php %}
# Tell views about your module
function your_module_views_api() {
    return array(
        'api' => 3,
        'path' => drupal_get_path('module', 'your_module') . '/inc',
    );
}

# Add your pseudo columns to the node table
function your_module_views_data() {
  $data['node']['column_name'] = array(
    'title' => t('Custom Column'),
    'help' => t('This is where your custom data will go.'),
    'field' => array(
      'handler' => 'custom_named_handler_field_custom_data',
    ),
  );

  return $data;
}

# Tell views about your custom handler
function your_module_views_handlers() {
  return array(
    'info' => array(
      'path' => drupal_get_path('module', 'your_module') . '/inc',
    ),
    'handlers' => array(
      'custom_named_handler_field_custom_data' => array(
        'parent' => 'views_handler_field',
      ),
    ),
  );
}

# Do your secondary query in hook_views_pre_render()
function your_module_views_pre_render(&$view) {
  if ($view->name == 'the_name_of_view_to_modify') {
    # Create custom property on the views object to store our external data
    $view->your_special_property = array();

    # Get the node id's of the views result set
    $nids = array();
    foreach($view->result as $r) {
      $nids[] = $r->nid;
    }
    
    # Switch to secondary database
    db_set_active('name_to_refer_to_database');

    # Query for secondary database data utilizing the nids you gathered from the views result set
    $query = db_select('table_name', 'tn');
    # Add whatever query stuff you need here like the following:
    $query->fields('tn', array('nid'));
   
    $results = $query->execute();
    foreach($results as $result) {
      $view->your_special_property[$result->nid]['custom_data'] = $result->custom_data;
    }
    # Switch back to the default database
    db_set_active('default');
  }
}
{% endhighlight %}

In your custom module's .info file you will need to add the handler class file so that it gets required

{% highlight bash %}
name = Your Custom Module
description = Module for adding additional data from a secondary database to a view.
core = 7.x
package = Your package
version = 7.x-1.0
files[] = inc/your_module_handler_field_custom_data.inc 
{% endhighlight %}

Finally, create your views handler.  It should probably extend from views_handler_field_entity as that is the parent class that other more custom views field classes derive from.  You could also extend one of the more specific child classes if you wanted to utilize and inherit some of their functionality.

{% highlight php %}
<?php
class your_module_handler_field_custom_data extends views_handler_field_entity {
  /** 
   * Overridden from parent
   */
  function render($values) {
    // make sure we have an array of the custom data we queried
    if (!empty($this->view->your_special_property[$values->nid]['custom_data'])) {
      return $this->view->your_special_property[$values->nid]['custom_data']
    } else {
      return 0;
    }   
  }
}
{% endhighlight %}

I hope the general concept of this views hacking procedure isn't too convoluted and that most people will find it understandable.  If anyone has any questions or knows of a cleaner more efficient way to accomplish what I did here please leave me a comment.
