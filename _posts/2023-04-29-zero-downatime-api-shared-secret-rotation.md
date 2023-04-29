---
layout: post
title: "Zero Downtime API Shared Secret Rotation"
description: 'A demonstration of how an API could rotate its shared secret with zero down time'
tags: [secret rotation, zero down time, shared secret]
---

# Zero Downtime API Shared Secret Rotation

Secrets are used in software applications for many different different things from connecting to a database to signing a JWT passed between services. It is considered a security best practice to rotate these types of secrets on a regular basis. Some reasons for this are:

- If a secret is accidentally exposed in logs, accidentally committed to version control, etc., having a process in place which smoothly rotates a secret turns what would be a firedrill into much less risky operation
- Rotating secrets regularly helps to ensure that if a malicious
  insider acquired a secret, that the time it would be viable would be minimized

It is common that applications will have secrets provided to them through
configuration or the environment and the secret will rarely change if ever. Some
secrets are harder to rotate than others but with a little planning, the tools
exist to make secret rotation possible and possibly even with zero downtime. AWS
provides some good documentation on retrieving secrets from [Secrets
Manager](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/secrets-app-secrets-manager.html) and [Parameter Store](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/secrets-app-ssm-paramstore.html) as well as some [examples of dynamic credential rotation](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/rotate-database-credentials-without-restarting-containers.html).
We can use this information to put together a solution for how we can rotation
credentials within our applications. Lets get started.

## A Shared Secret

Lets suppose we have application A and application B. In the spirit of
zero-trust we aren't going to just let the applications trust the network and
communicate without any form authentication or authorization. Instead, the
services will utilize a shared secret. This could be used for a JWT HMAC or
simply a token used for bearer authentication. To keep things simple, we'll use
bearer authentication in this demonstration.

## Storing and Sharing the Secret

For this example we will use AWS SSM for storing the shared secret. This could
also be AWS secrets manager, GCP secret manager, Azure key vault, etc. The
important thing is that the secret is stored centrally in a secrets manager and
the applications have the appropriate privileges so that they can access it. The
`SecureString` type is used to ensure that the secret is encrypted with KMS and
we set the secret to `MyInitialSecret`.

![Initial AWS SSM Parameter](/assets/SecretRotation/InitialAWSSSMParameter.png)

## The API

The first thing we will do is create a small api that has an endpoint protected with
bearer authentication. The secret is acquired from the environment the way most
applications typically would:

{% highlight python %}
import os
from flask import Flask
from flask_httpauth import HTTPTokenAuth

app = Flask(__name__)
auth = HTTPTokenAuth(scheme='Bearer')
s = Secrets()

@auth.verify_token
def verify_token(token):
    if token == os.getenv('SECRET_TOKEN'):
        return True

@app.route('/')
@auth.login_required
def hello():
    return 'Hello, World!'
{% endhighlight %}

Next, we'll create a class that:

- Has a method to pull / refresh credentials from AWS Parameter Store
- Has a method to get a secret from the object
- Acquires AWS parameter store credentials when the object is instantiated

{% highlight python %}
class Secrets:
    # Pull the secrets from Parameter Store when object instantiated
    def refresh(self):
        print("Retrieving secrets from Parameter Store")
        ssm = boto3.client('ssm', 'us-east-1')
        response = ssm.get_parameters(
                Names=['RotationExample'],WithDecryption=True
        )

        for parameter in response['Parameters']:
            self.secrets[parameter['Name']] = parameter['Value']

    # Method to get a secret from the object
    def secret(self, key):
        return self.secrets[key]

    # Acquire the secrets from AWS when the object is instantiated
    def __init__(self):
        self.secrets = {}
        self.refresh()
{% endhighlight %}

Now, we'll update the api code from before to instantiate an instance of this
class when the app boots up to make the token comparison to the secret stored in
the object instead of from the environment:

{% highlight python %}
.. snip ..

app = Flask(__name__)
auth = HTTPTokenAuth(scheme='Bearer')
s = Secrets()

@auth.verify_token
def verify_token(token):
    if token == s.secret('RotationExample'):
        print(f'Token match!')
        return True
    else:
        return False

.. snip ..

{% endhighlight %}

And lastly we'll update the code so that when a client authenticates the api
first tries to authenticate against the secret the app acquired at boot. If that
fails, the app will use the `refresh` method on the `Secrets` object to update
the secrets from Parameter Store and then attempt the comparison again:

{% highlight python %}
.. snip ..

@auth.verify_token
def verify_token(token):
    # Try the secret we have in memory
    if token == s.secret('RotationExample'):
        print(f'Token match!')
        return True
    else:
    # If it fails, go refresh secrets from Parameter Store
        print(f'Token mis-match. Refreshing token')
        s.refresh()
        if token == s.secret('RotationExample'):
            return True
    return False

.. snip ..
{% endhighlight %}

Putting it all together, the final code looks like this:

{% highlight python %}

import boto3
from flask import Flask
from flask_httpauth import HTTPTokenAuth

class Secrets:
    # Pull the secrets from Parameter Store when object instantiated
    def refresh(self):
        print("Retrieving secrets from Parameter Store")
        ssm = boto3.client('ssm', 'us-east-1')
        response = ssm.get_parameters(
                Names=['RotationExample'],WithDecryption=True
        )

        for parameter in response['Parameters']:
            self.secrets[parameter['Name']] = parameter['Value']

    # Method to get a secret from the object
    def secret(self, key):
        return self.secrets[key]

    # Acquire the secrets from AWS when the object is instantiated
    def __init__(self):
        self.secrets = {}
        self.refresh()

app = Flask(__name__)
auth = HTTPTokenAuth(scheme='Bearer')
s = Secrets()

@auth.verify_token
def verify_token(token):
    # Try the secret we have in memory
    if token == s.secret('RotationExample'):
        print(f'Token match!')
        return True
    else:
    # If it fails, go refresh secrets from Parameter Store
        print(f'Token mis-match. Refreshing token')
        s.refresh()
        if token == s.secret('RotationExample'):
            return True
    return False

@app.route('/')
@auth.login_required
def hello():
    return 'Hello, World!'
{% endhighlight %}

Alright! Lets give it a shot!

We'll boot up our Flask api:

{% highlight bash %}
export FLASK_APP=hello
export FLASK_ENV=development

╰─❯ flask run
Retrieving secrets from Parameter Store
 * Serving Flask app 'hello'
 * Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on http://127.0.0.1:5000
Press CTRL+C to quit
{% endhighlight %}

In the output we see the "Retrieving secrets from Parameter Store" output so
presumeabley the secrets are pulled correctly when the app boots. Now we'll use curl to make a call to our api with the initial secret we set of `MyInitialSecret` to test things out:

{% highlight bash %}
curl -H 'Authorization: Bearer MyInitialSecret' http://127.0.0.1:5000
Hello, World!%
{% endhighlight %}

The call succeeds as we would expect as the app pulled the secret when it booted
and it matches the secret we send in the authorization header. In the output on
the server side we see:

{% highlight bash %}
Token match!
{% endhighlight %}

Now for the magic. We will go to Parameter Store and update the secret to a new
value of `MyNewSecret`:

![Updated AWS SSM Parameter](/assets/SecretRotation/UpdatedAWSSSMParameter.png)

And without restarting our server (cause that would be cheating ;)) we use curl
again with the updated secret:

{% highlight bash %}
╰─❯ curl -H 'Authorization: Bearer MyNewSecret' http://127.0.0.1:5000
Hello, World!%
{% endhighlight %}

The request succeeds with the new secret! Lets look at the server logs. We see:

{% highlight bash %}
Token mis-match. Refreshing token
Retrieving secrets from Parameter Store
127.0.0.1 - - [29/Apr/2023 18:01:00] "GET / HTTP/1.1" 200 -
{% endhighlight %}

So the server detected the token mis-match, it refreshed the secrets and
revalidated the token. The token validated successfully and allows the request
to succeed.

And just to prove we don't have anything up our sleeve, we'll make another
request with a bogus secret:

{% highlight bash %}
╰─❯ curl -H 'Authorization: Bearer FakeSecret' http://127.0.0.1:5000
Unauthorized Access%
{% endhighlight %}

And as we would expect, we are denied access.

## Conclusion

I hope this blog post helps to make secret rotation a little less intimidating
and more approachable for engineers. This solution does have its weaknesses. One weakness is that every failed token validtion triggers a call to AWS to refresh to token. This may or may not be an issue for an internal api but with a service like secrets manager that charges per api call, it might not be something you want someone to pound on and run up your bills. To address this the code could be updated to be more robust and implement a ttl of time between refreshes, that way the app would only attempt to refresh every so often instead constantly.

This post has been focused on the api server side of the secret rotation but how
might the client get updated? It would actually be very similar to how the
server handles the rotation but this post has become pretty lengthy so I'll save
that for my next post.
