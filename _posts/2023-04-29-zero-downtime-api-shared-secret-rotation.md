---
layout: post
title: "Zero Downtime API Shared Secret Rotation"
description: 'A demonstration of how an API could rotate its shared secret with zero down time'
tags: [secret rotation, zero down time, shared secret]
---

# Zero Downtime API Shared Secret Rotation

Secrets are used in software applications for many different things from connecting to a database to signing a JWT passed between services. It is considered a security best practice to rotate these types of secrets on a regular basis. Some reasons for this are:

- If a secret is accidentally exposed in logs, accidentally committed to version control, etc., having a process in place which smoothly rotates a secret turns what would be a firedrill into much less risky operation
- Rotating secrets regularly helps to ensure that if a malicious
  insider acquired a secret, that the time it would be viable would be minimized

It is common that applications will have secrets provided to them through configuration or the environment and the secret will rarely change if ever. Some secrets are harder to rotate than others but with a little planning, the tools exist to make secret rotation possible and possibly even with zero downtime. AWS provides some good documentation on retrieving secrets from [Secrets Manager](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/secrets-app-secrets-manager.html) and [Parameter Store](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/secrets-app-ssm-paramstore.html) as well as some [examples of dynamic credential rotation](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/rotate-database-credentials-without-restarting-containers.html).  We can use this information to put together a solution for how we can rotate credentials within our applications. Lets get started.

## A Shared Secret

Lets suppose we have application A and application B. In the spirit of zero-trust we aren't going to just let the applications trust the network and communicate without any form authentication or authorization. Instead, the services will utilize a shared secret. This could be used for a JWT HMAC or simply a token used for bearer authentication. To keep things simple, we'll use bearer authentication in this demonstration.

## Storing and Sharing the Secret

For this example we will use AWS SSM Parameter Store for storing the shared secret. This could also be AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, etc. The important thing is that the secret is stored centrally in a secrets manager and the applications have the appropriate privileges so that they can access it.

_NOTE: AWS Parameter Store and Secrets manager each have their own pluses and minuses so read up on the differences to select the one which fits your use case best._

We'll use the `SecureString` type when configuring the parameter to ensure that the secret is encrypted with KMS and we set the secret to `MyInitialSecret`.

![Initial AWS SSM Parameter](/assets/SecretRotation/InitialAWSSSMParameter.png)

## The API

The first thing we will do is create a small Python Flask api that has an endpoint protected with bearer authentication. The secret is acquired from the environment the way most applications typically would:

{% highlight python %}
import os
from flask import Flask
from flask_httpauth import HTTPTokenAuth

app = Flask(__name__)
auth = HTTPTokenAuth(scheme='Bearer')

@auth.verify_token
def verify_token(token):
    if token == os.getenv('SECRET_TOKEN'):
        return True

@app.route('/')
@auth.login_required
def main():
    return 'Open Sesame!'
{% endhighlight %}

Next, we'll create a `SecretsCache` class that:

- Has a method to pull / refresh credentials from AWS Parameter Store
- Has a method to get a secret from the object
- Acquires AWS parameter store credentials when the object is instantiated
- Has a ttl (time to live) feature to automatically refresh secrets when they become stale
- Leverages "current" and "previous" versions of secrets to allow for rolling updates

{% highlight python %}
import boto3
from datetime import datetime, timedelta

class SecretsCache:
    # Acquire the secrets from AWS when the object is instantiated
    def __init__(self, keys, region, ttl=1):
        # Initialize a dictionary for each secret key name
        self.secrets    = {}
        for key in keys:
            self.secrets[key] = {}

        self.region     = region
        # Set a 1 minute ttl on refreshing secrets
        self.ttl        = ttl
        self.updated_at = None

        # Fetch the secrets
        self.refresh()

    # Pull the secrets from Parameter Store
    def refresh(self):
        print("Retrieving secrets from Parameter Store")

        # Get current and previous versions of each secret
        for key in self.secrets:
            self.secrets[key] = self.__get_secret_versions(key)

        # Set the updated_at property to now
        self.updated_at = datetime.now()

    # Method to get a secret from the object
    # Default to getting the Current secret
    def secret(self, key, version='Current'):
        # Check if our secrets need a refresh on access
        if self.__stale_secrets():
            self.refresh()

        return self.secrets[key][version]

    def __stale_secrets(self):
        if datetime.now() > self.updated_at + timedelta(minutes = self.ttl):
            print("TTL expired!")
            return True
        else:
            return False

    # Looks up current and previous version for a secret
    def __get_secret_versions(self, key):
        values = {'Current': None, 'Previous': None}

        ssm = boto3.client('ssm', self.region)

        response = ssm.get_parameter(
                Name=key,WithDecryption=True
        )

        if response and response['Parameter']['Value']:
            values['Current'] = response['Parameter']['Value']

            # Parameter store increments the version each time a secret is
            # updated and it starts with 1 for the first secret
            # Because of this we only lookup a previous version if the secret is
            # the second version or more
            if response['Parameter']['Version'] > 1:
                response = ssm.get_parameter(
                        Name=f'{key}:{(response["Parameter"]["Version"]-1)} ',
                        WithDecryption=True
                )

                if response and response['Parameter']['Value']:
                    values['Previous'] = response['Parameter']['Value']

        return values
{% endhighlight %}

The reason for the ttl in the class is that if the server is never rebooted it will never know that a secret has been rotated. It needs something to wake it up and refresh to secrets. The reason for fetching and using current and previous versions of a secret is that in a production environment not all clients and servers will necessarily recive updated credentials at the same time. By having both previous and current secrets the server can support a client that hasn't received the updated secret yet.

Now, we'll update the api code from before to instantiate an instance of this class when the app boots up to make the token comparison to the secret stored in the object instead of from the environment. We'll also update the token validation to try the current secret, try the previous secret, and if both fail, refresh and retry the current secret:

{% highlight python %}
.. snip ..
import secretscache as sc

# Initialize some globals
api_token_key = 'RotationExample'
aws_region    = 'us-east-1'

app = Flask(__name__)
auth = HTTPTokenAuth(scheme='Bearer')

# Load up our secrets on boot
s = sc.SecretsCache([api_token_key], aws_region)

@auth.verify_token
def verify_token(token):
    print(f'Using token: {token} for demonstration purposes only. Don\'t do this in production!')
    # Try the most current secret we have in memory
    if token == s.secret(api_token_key):
        print(f'Current token match!')
        return True
    elif token == s.secret(api_token_key, 'Previous'):
        print(f'Previous token match!')
        return True
    else:
        # If current and previous fail, go refresh secrets from Parameter Store
        # Retry current which may now be new if it was updated
        print(f'Token mis-match. Refreshing token')
        s.refresh()
        if token == s.secret(api_token_key):
            return True
    return False

.. snip ..

{% endhighlight %}

The reason after we try current and previous secrets we refresh and retry current again is it could be possible that a client has received an updated secret before the server did. In that case neither the current or previous secrets would match the new secret the client holds. By refreshing the secrets on the server at this point, the server should receive the newly updated secret which becomes the new current and the request should succeed.

Putting it all together, the final code looks like this:

`secretscache.py`

{% highlight python %}
import boto3
from datetime import datetime, timedelta

class SecretsCache:
    # Acquire the secrets from AWS when the object is instantiated
    def __init__(self, keys, region, ttl=1):
        # Initialize a dictionary for each secret key name
        self.secrets    = {}
        for key in keys:
            self.secrets[key] = {}

        self.region     = region
        # Set a 1 minute ttl on refreshing secrets
        self.ttl        = ttl
        self.updated_at = None

        # Fetch the secrets
        self.refresh()

    # Pull the secrets from Parameter Store when object instantiated
    def refresh(self):
        print("Retrieving secrets from Parameter Store")

        # Get current and previous versions of each secret
        for key in self.secrets:
            self.secrets[key] = self.__get_secret_versions(key)

        # Set the updated_at property to now
        self.updated_at = datetime.now()

    # Method to get a secret from the object
    # Default to getting the Current secret
    def secret(self, key, version='Current'):
        # Check if our secrets need a refresh on access
        if self.__stale_secrets():
            self.refresh()

        return self.secrets[key][version]

    def __stale_secrets(self):
        if datetime.now() > self.updated_at + timedelta(minutes = self.ttl):
            print("TTL expired!")
            return True
        else:
            return False

    # Looks up current and previous version for a secret
    def __get_secret_versions(self, key):
        values = {'Current': None, 'Previous': None}

        ssm = boto3.client('ssm', self.region)

        response = ssm.get_parameter(
                Name=key,WithDecryption=True
        )

        if response and response['Parameter']['Value']:
            values['Current'] = response['Parameter']['Value']

            # Parameter store increments the version each time a secret is
            # updated and it starts with 1 for the first secret
            # Because of this we only lookup a previous version if the secret is
            # the second version or more
            if response['Parameter']['Version'] > 1:
                response = ssm.get_parameter(
                        Name=f'{key}:{(response["Parameter"]["Version"]-1)} ',
                        WithDecryption=True
                )

                if response and response['Parameter']['Value']:
                    values['Previous'] = response['Parameter']['Value']

        return values

{% endhighlight %}

`server.py`:
{% highlight python %}
from flask import Flask
from flask_httpauth import HTTPTokenAuth
import secretscache as sc

# Initialize some globals
api_token_key = 'RotationExample'
aws_region    = 'us-east-1'

# Initialize a Flask app 
app = Flask(__name__)
auth = HTTPTokenAuth(scheme='Bearer')

# Load up our secrets on boot
s = sc.SecretsCache([api_token_key], aws_region)

@auth.verify_token
def verify_token(token):
    print(f'Using token: {token} for demonstration purposes only. Don\'t do this in production!')
    # Try the most current secret we have in memory
    if token == s.secret(api_token_key):
        print(f'Current token match!')
        return True
    elif token == s.secret(api_token_key, 'Previous'):
        print(f'Previous token match!')
        return True
    else:
        # If current and previous fail, go refresh secrets from Parameter Store
        # Retry current which may now be new if it was updated
        print(f'Token mis-match. Refreshing token')
        s.refresh()
        if token == s.secret(api_token_key):
            return True
    return False

@app.route('/')
@auth.login_required
def main():
    return 'Open Sesame!'
{% endhighlight %}

Alright! Lets give it a shot! Let's boot up our Flask api:

{% highlight bash %}
export FLASK_APP=server
export FLASK_ENV=development

╰─❯ flask run
Retrieving secrets from Parameter Store
 * Serving Flask app 'server'
 * Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on http://127.0.0.1:5000
Press CTRL+C to quit
{% endhighlight %}

In the output we see the "Retrieving secrets from Parameter Store" output so presumeabley the secrets are pulled correctly when the app boots. Now we'll use curl to make a call to our api with the initial secret we set of `MyInitialSecret` to test things out:

{% highlight bash %}
curl -H 'Authorization: Bearer MyInitialSecret' http://127.0.0.1:5000
Open Sesame!%
{% endhighlight %}

The call succeeds as expected as the app pulled the secret when it booted and it matches the secret we send in the authorization header. In the output on the server side we see:

{% highlight bash %}
Using token: MyInitialSecret for demonstration purposes only. Don't do this in production!
Current token match!
{% endhighlight %}

Now for the magic. We will go to Parameter Store and update the secret to a new value of `MyNewSecret`:

![Updated AWS SSM Parameter](/assets/SecretRotation/UpdatedAWSSSMParameter.png)

And without restarting our server (because that would be cheating ;)) we use curl again with the updated secret:

{% highlight bash %}
╰─❯ curl -H 'Authorization: Bearer MyNewSecret' http://127.0.0.1:5000
OpenSesame!%
{% endhighlight %}

The request succeeds with the new secret! Lets look at the server logs. We see:

{% highlight bash %}
Using token: MyNewSecret for demonstration purposes only. Don't do this in production!
TTL expired!
Retrieving secrets from Parameter Store
Previous token match!
127.0.0.1 - - [29/Apr/2023 18:01:00] "GET / HTTP/1.1" 200 -
{% endhighlight %}

The output may differ depending on if the secret is rotated and curl executed before the ttl has expired or not. In this case the ttl had expired so when the new secret was tried the server pulled the new secrets and client secret against the refreshed secret which match so the request succeeds.

And just to prove we don't have anything up our sleeve, we'll make another request with a bogus secret:

{% highlight bash %}
╰─❯ curl -H 'Authorization: Bearer FakeSecret' http://127.0.0.1:5000
Unauthorized Access%
{% endhighlight %}

And as we would expect, we are denied access.

## What About The Client

You might wonder what the client side code might look like to support the secret
rotation. In actuality the client side code could simply use a class like the
`SecretsCache` class as is. For example:

`client.py`
{% highlight python %}
import requests
import secretscache as sc
import time

# Initialize some globals
api_token_key = 'RotationExample'
aws_region    = 'us-east-1'

api_endpoint='http://localhost:5000/'
s = sc.SecretsCache([api_token_key], aws_region)

# Loop forever
while True:
    print(f'Trying to access api using secret: {s.secret(api_token_key)}. For demonstration purposes only. Don\'t log secrets!')
    r = requests.get(api_endpoint, headers={"Authorization": f'Bearer {s.secret(api_token_key)}'})
    print(f'Request was {r.status_code}')
    time.sleep(5)

if __name__ == '__main__':
    sys.exit(main())
{% endhighlight %}

Here we have a simple Python script that makes a request in a loop every 5 seconds. The script leverages our `SecretsCache` class again but needs no other special code to support the secret rotation. The class has the ttl logic built into it so it will refresh the secrets every so often. The client doesn't need to be aware of current and previous versions so long as the server handles it, which it is doing in our example. We can start up our server and client and rotate the secret in AWS and when the ttl expires, we can see the magic happen:

In the client console when we rotate the secret, we see:

{% highlight bash %}
.. snip ..

Trying to access api using secret: Abracadabra. For demonstration purposes only. Don't log secrets!
Request was 200
Trying to access api using secret: Abracadabra. For demonstration purposes only. Don't log secrets!
Request was 200
TTL expired!
Retrieving secrets from Parameter Store
Trying to access api using secret: AlaKazaam!. For demonstration purposes only. Don't log secrets!
Request was 200
Trying to access api using secret: AlaKazaam!. For demonstration purposes only. Don't log secrets!
Request was 200

.. snip ..
{% endhighlight %}

And in the server console we see:

{% highlight bash %}
.. snip ..

Current token match!
127.0.0.1 - - [06/May/2023 21:08:57] "GET / HTTP/1.1" 200 -
Using token: Abracadabra for demonstration purposes only. Don't do this in production!
Current token match!
127.0.0.1 - - [06/May/2023 21:09:02] "GET / HTTP/1.1" 200 -
Using token: Abracadabra for demonstration purposes only. Don't do this in production!
TTL expired!
Retrieving secrets from Parameter Store
Previous token match!
127.0.0.1 - - [06/May/2023 21:09:07] "GET / HTTP/1.1" 200 -
Using token: AlaKazaam! for demonstration purposes only. Don't do this in production!
Current token match!
127.0.0.1 - - [06/May/2023 21:09:13] "GET / HTTP/1.1" 200 -
Using token: AlaKazaam! for demonstration purposes only. Don't do this in production!
Current token match!

.. snip ..
{% endhighlight %}

We can see that after rotation the secret from `Abracadabra` to `AlaKazaam!`, when the ttl expires on the client, the new secret is picked up and tried. On the server side, the ttl has expired so the new secret is picked up and compared with `AlaKazaam!` which matches so the request suceeds.

## Conclusion

I hope this post has helped to make secret rotation a little less intimidating and more approachable for people. The`SecretsCache` class described in the post can easily be modified to work with AWS Secrets Manager or some other cloud providers secrets management service. In our example code we have used a very small ttl default of 1 minute but you likely wouldn't need to refresh the secrets this frequently. This was simply done to speed up my demonstration. Somewhere between 10 - 60 minutes might be a more reasonable default to allow you to rotate the secret in case of exposure and have it rolled over on all servers in a reasonable amount of time. The sample code from the post can be found [over on my GitHub](https://github.com/blischalk/SharedSecretRotation) Happy secret rotating!
