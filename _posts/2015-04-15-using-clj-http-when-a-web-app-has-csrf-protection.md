---
layout: post
title: "Using Clj-http When A Web App Has CSRF Protection"
description: "Recently when working on a Clojure web app I ran into a scenario where I wanted to do a little bit of integration testing. What I wanted to do was post some data to the same endpoint that my webform would post to. The problem was that the app used ring-anti-forgery for CSRF protection. Because of this, I needed to figure out a way to spoof this anti-forgery token when making POST requests using Clj-http. The solution came out of a lot of trial and error."
redirect_from:
  - /posts/17-using-clj-http-when-a-web-app-has-csrf-protection
tags: [clj-http, clojure, ring]
---
Recently when working on a Clojure web app I ran into a scenario where I wanted to do a little bit of integration testing. What I wanted to do was post some data to the same endpoint that my webform would post to. The problem was that the app used [ring-anti-forgery](https://github.com/ring-clojure/ring-anti-forgery) for CSRF protection. Because of this, I needed to figure out a way to spoof this anti-forgery token when making POST requests using [Clj-http](https://github.com/dakrone/clj-http). The solution came out of a lot of trial and error.

The first problem was that I needed to figure out a way to acquire the anti-forgery token. I did this by doing a GET request to the page that would contain the form that I wanted to spoof. I grabbed the body of the result of the GET request and used a regex to find and scrape the token out of the HTML.

The second task was sending the CSRF token along with my posted data. This seemed problematic at first because of the next issue that I ran into but later realized that it isn't that difficult. To send along the CSRF token, all that needs to be done is to either send it as a normal form param named "\_\_anti-forgery-token" or it can also be passed in a header as either "X-CSRF-Token" or "X-XSRF-Token".

I thought this was all that would be needed to get my form spoofing working from clj-http but at this point things were not working. Ring kept complaining that the CSRF token was still invalid. It took me quite a bit to track down what I was missing but eventually I determined that ring-anti-forgery uses your ring session to match up the anti-forgery token that is being sent with the user who is making the request. So now the last task was to figure out how to send along the ring-session.

The solution here turned out to be using a "cookie-store". Before making the first request to acquire the CSRF token I had to initialize a cookie-store. I then would pass that along with all the future requests that I would make. By doing this the cookie store would get updated and read just as a users session would when interacting with the browser. On the first request for the token the cookie-store gets the ring-session stored in it. In the request to post the params ring is then able to compare the session with the CSRF token and verify that it is correct making the whole process complete.

The whole process looked a little something like this:

{% highlight clojure %}
(let [cookie-store (clj-http.cookies/cookie-store)
      r1 (http/get "http://localhost/foo"
           {:throw-exceptions false
            :cookie-store cookie-store})
      token-regex (str "name=\"__anti-forgery-token\" "
                       "type=\"hidden\" value=\"(.+?)\"")
      token (-> token-regex
                re-pattern
                (re-find (:body r1))
                         second)
      r2 (http/post "http://localhost/bar"
                    {:headers {"X-CSRF-Token" token}
                     :form-params {:foo "bar"}
                     :throw-exceptions false
                     :cookie-store cookie-store})])
{% endhighlight %}



