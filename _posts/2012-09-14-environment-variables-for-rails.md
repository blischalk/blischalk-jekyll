---
layout: post
title: "Environment Variables for Rails"
description: "For security purposes I like to keep confidential information out of my git repositories. While hosting a site with Heroku for a little while I learned that they advocate environment variables for storing sensitive information. I later switched over to my own server and wanted to replicate this setup. I threw my environment variables in the .bash_profile of the user associated with my application and found out that when ruby is started on the server a login doesn’t appear to occur for that user even though the process is owned by the user. The .bash_profile never appears to get loaded up. My environment variables weren’t being loaded up into my application as I had anticipated."
redirect_from:
  - /posts/11-environment-variables-for-rails
tags: [environment, variables, ruby]
---

For security purposes I like to keep confidential information out of my git repositories.  While hosting a site with Heroku for a little while I learned that they advocate environment variables for storing sensitive information.  I later switched over to my own server and wanted to replicate this setup.  I threw my environment variables in the .bash_profile of the user associated with my application and found out that when ruby is started on the server a login doesn't appear to occur for that user even though the process is owned by the user.  The .bash_profile never appears to get loaded up.  My environment variables weren't being loaded up into my application as I had anticipated.  

After a lot of research on the internet I found that RVM has a file for each of your gemsets located at /usr/local/rvm/environments.  In this file environment variables are setup for your ruby environment including your RUBY_VERSION, GEM_HOME, etc.  I added my desired environment variables to the gemset and that seemed to do the trick.  All of the information I wanted to pass to my ruby app was available to my code.  I'm not sure if this is the best way to go about passing environment variables to your ruby application but it seemed to be the only thing that worked for me.  Note: On Nginx you will need to service nginx reload to have the environment variables take effect

If anyone else has alternative methods please feel free to comment.
