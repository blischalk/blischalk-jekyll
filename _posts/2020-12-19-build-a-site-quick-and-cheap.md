---
layout: post
title: "Build A Web Site/App Quick and Cheap!"
description: 'Cheapest and fastest ways to get your project online'
tags: [thrifty, cheap, fast, webapp, php, ruby, static site generator, gitlab, github]
---

# Build A Web Site/App Quick and Cheap!

Recently I needed to make some updates to a Rails app I built many years ago for
a friend. It is deployed on Heroku, uses S3 for file uploads, and
costs ~$7 a month on Heroku's Hobby plan. While working on it a couple things
occurred to me:

1. I love Platform as a Service (PaaS) for deploying applications so that I
don't have to deal with setup and maintenance of infrastructure. OS level updates, etc.
2. $7 a month still seems sort of expensive for a brochure type site that
   someone only logs in to update a few times a year.

This got me wondering if you could deploy a basic web app for cheaper than
Heroku's offering and what sort of sacrifices you might have to make to achieve
it. Now I know you'll all mention the hack to use Heroku's Free tier and then
use New Relic or Pingdom to keep your app awake but Heroku got wise to this and
forces apps to sleep after exhausting free hours. One of the criteria I am
defining here is that no tom foolery should be involved with running the app
cheaply!

## Potential Options

### Static Site Generators on GitLab / GitHub / Cloudflare Pages

In doing some research on the topic, the absolute cheapest and fastest option would be a
static website deployed on [GitLab](https://docs.gitlab.com/ee/user/project/pages/) or [GitHub pages](https://pages.github.com/).

_NOTE: Cloudflare also recently entered the static website market, also adding something called workers for added backend functionality, called [Cloudflare Pages](https://pages.cloudflare.com/). Their offering is also free_ These are completely free,
support static site generators, offer ssl certificates, CI/CD pipelines, etc.

_ANOTHER NOTE: [Amazon
S3](https://aws.amazon.com/getting-started/hands-on/host-static-website/) is
another low price option for hosting a static website however, beyond the 12
months free tier, you would likely be spending a couple of dollars a month
which, it seems like the free GitLab / GitHub / Cloudflare pages would be the
better opton given it is free, provides version control, and provides CI/CD
pipelines._

Really GitLab / GitHub is an awesome value and is my #1 choice for deploying a site for
cheap. The down side here is that is a static web site and not a web app (though
Cloudflare Pages seems to intend to change that). If you are trying to build something for a client who then wanted additonal functionality; content management, etc. then this option may start to break down and you may need something that provides the ability for backend development. This leads us to the next option...

### Amazon Lightsail and Bitnami

The next cheapest and fastest option I could find is [Amazon
Lightsail](https://aws.amazon.com/lightsail/) and [Bitnami
images](https://bitnami.com/stacks).
Amazon Lightsail can deploy the smallest VPS instance for $3.50 a month and run
Bitnami images of Wordpress, Drupal, Node.js, and various other stacks. The
great thing about Bitnami images is that they come pretty much pre-configured
for you to run your app. Turnkey VPS for your app technology of choice. The one
down side I found is that it doesn't seem that there is a Bitnami image for
deploying Ruby on Rails so you would likely have to build out that stack
yourself to host a Rails app; which, is sort of a bummer.

The other down side of this option is that there is now infrastructure to maintain. Most of the configuration / installation of packages and such is taken care of so you do reap the benefits of that. However, when there are vulnerabilities in mysql, nginx, etc. or OS updates are necessary, it will still be on the user to take care of it. After thinking about this though it seems like it may not be such a big deal. In the spirit of "Cattle, Not Pets", you could likely manage the provisioning of new Amazon Lightsail images via Terraform and code deployment via Ansible, Capistrano, Mia, or just some shell scripts. It wouldn't be as hands off as Heroku, but could likely be as simple as provision a new server and deploy the app, then swap the DNS.

### Digital Ocean $5 Droplet

Getting a little more expensive I came across the [$5 droplet from Digital
Ocean](https://www.digitalocean.com/products/droplets/). Digital Ocean has
one-click installs of common stacks / frameworks similar to AWS Lightsail /
Bitnami.

### GoDaddy, BlueHost, AppGator, A2Hosting, Etc.

There are obviously other deals out there with some of the other hosting
companies like GoDaddy and such but you start to encounter things like:

- Needing to sign multi-year contracts to get the cheap price
- Price increase after renewal
- Access restrictions to the server
- Needing to install / configure things yourself

### An Aside: Only For Small Stuff

Obviously this fast and cheap approach I am researching here is really only
applicable to very small projects. I see this as being a solution for the
freelancer building brochure sites for small businesses who may need some back
end programming logic but aren't needing Elasticsearch clusters,
multi-availability zone auto-scaling, message queue infrastructure, etc.

### Another Aside: Openshift

I remember using Red Hat Openshift for a project long ago and remember / have
read that its priced pretty low. However, I had some difficulty revisiting it
recently. Apparently my account is in a weird state that needs a "complete profile" which, when I complete it, it sends me back to do it all over again. Soooo... Not going to explore that until they sort that out.

### One More: Amazon AWS Free Tier

Someone will also invariably offer the Amazon Free tier as an opiton for cheap
hosting. However, this is only available for 12 months for EC2, (I've been
using AWS for much longer than this so that ship has sailed.) Someone may also
mention just creating new / multiple accounts, etc. but again then, we're back
to tom foolery.

Amazon AWS also offers some "Always Free Tier" options such as Lambda,
Dynamo DB, Amazon SNS, etc. but I haven't really explored it. It seems like you
could likely add some back-end functionality to your static site for free /
cheap through Lamda and Dynamo DB so if that is something you may be interested
in, you can find more information on Amazon AWS free tier services
[here](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc).

### Language / Frameworks

I noticed that it seems that the most common type of web site e.g Wordpress,
Drupal, etc. and PHP as a language, have the most options for easy
deployment with minimal configuration for cheap. It seems like Java, Ruby, etc. cost more unless you are willing to install and configure things
yourself. For example, you can easily deploy a Java war file on Amazon
Elasticbeanstalk almost as easily as Heroku. However, when you add in an RDS database instance you are going to be
looking at at least $25 a month to run the app. When compared to a Drupal
one-click install, or a PHP Laravel app running on a LAMP stack, programming
languages and frameworks beyond PHP don't seem to have cheap hosting options
like PHP based apps do.

### Conclusion

So that pretty much rounds out the options I've found for cheap hosting.
Hopefully you've found this helpful and learned some ways to save yourself some
money hosting small sites for yourself or your clients.
