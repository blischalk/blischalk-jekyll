---
layout: post
title: "Lets Build A CI Pipeline Threat Model"
description: 'Lets build a threat model of a CI pipeline for fun'
tags: [threat modeling, CI Pipeline, container registry]
---

# Lets Build A CI Pipeline Threat Model

I really enjoy threat modeling and thinking about all the ways things can go
wrong in a system. For those unfamiliar with it, the general idea is to first
determine:

## What Are We Building? (Draw A Picture)

When threat modeling the first step is to get clarity around what is being
built. A common way to achieve this is by drawing a picture. Architecture
diagrams, dataflow diagrams, attack tree diagrams, etc. can all be used and if
they already exist, are good starting points and save the work of having to draw
new ones from scratch. With that said, what are we building?

Today we will be building out a CI pipeline that allows an engineer to push
their code to a git version control system, which will then build, test, and
deploy a docker image to a container registry. To illustrate this process, I have put
together the following diagram:

![Container build, test, publish piepline](/assets/ci-threat-model.png)

## What Could _Possibly_ Go Wrong? ┌( ಠ‿ಠ )┘

Now that we know what we're building, we need to think about what could go
wrong? What might an attacker be able to do that could cause things to go
sideways? To help us think about potential issues there is the acronym STRIDE. Developed by
Microsoft, STRIDE helps us think about ways that security issues may arise in a
system by thinking about different types of security issues. The acronym stands
for:

* Spoofing
* Tampering
* Repudiation
* Information Disclosure
* Denial of Service
* Elevation of Privilege

When thinking about potential threats to a system the particular STRIDE categorization is
really un-important. Whether something is tampering or spoofing
really doesn't matter. For example, one could make the case that altering an ip
address in a request could be tampering or spoofing. The point of STRIDE is really to
provide a catalyst to get people thinking.

There are some other catalysts though that I find useful beyond STRIDE:

* Thinking about authentication and authorization
* Thinking about different kinds of threat actors e.g nation-state,
  angry former employee, hacktivist, etc.
* Thinking about types of vulnerabilities: SSRF, XSS, Insecure Deserialization,
  Insecure Configuration, etc.
* Assuming things such as an api key has already been compromised or a firewall
  has been misconfigured
* Thinking about other high-profile breaches and how those happened
* Thinking about red-teaming / pentesting experiences, hackthebox and other
  pentest training

So now that we know what we're building and have some things to prompt our
creativity, what could possibly go wrong in our CI pipeline deployment?

### Threats and Mitigations

After spending some time looking over the diagram and thinking about potential
threats, here are some that I came up with. For each one I also attempted to
think about some potential mitigations that people may choose to implement to
address the potential risk.

#### Threat

>> A developers laptop is lost or stolen. The hard drive is not encrypted so the attacker is able to use a tool such as [Konboot](https://kon-boot.com) to bypass password authentication and have commit access the all repositories to and ssh access to all systems the developer has access to.


#### Mitigations

* Ensure all company laptops are encrypted
* Ensure developers repository access follows the principle of least privilege
  and only has access to what they need to
* Ensure devices are using enterprise management software like Jamf, etc.
* Ensure all devices are attached to the domain
* Ensure all devices are running end point protection like [Crowdstrike](https://www.crowdstrike.com/)


#### Threat

>> A developer commits AWS secrets to version control either accidentally or on
>> purpose. An angry employee with read access to the repository finds the
>> secrets and proceeds to use them to steal end-user data.


#### Mitigations

* Use something like [git-secrets](https://github.com/awslabs/git-secrets) in
  repositories to help detect, flag, and block merge requests which may contain
  secrets
* Ensure the company is using secure secret management techniques such as tools
  like [Hashicorp Vault](https://www.vaultproject.io/)
* Ensure secrets are stored encrypted at rest
* Adhere to the principal of "Need to Know" and ensure that produciton secrets
  are not shared with individuals who don't have a need to know
* Adhere to the principal of "Least Privilege" and ensure secrets only have the
  the permissions to fulfill their use case
* Ensure logging is enabled so that secret usage can be tracked to the system
  and/or user who acquired and used them
* Enable and use threat detection such as [GuardDuty](https://aws.amazon.com/guardduty/) to detect anomolies in patterns of service operation
* Ensure version control system is only available on the corporate VPN


#### Threat

>> A developer creates a benign looking and named, but yet malicious package and hosts it on a public GitHub repo.
>> While working on a feature, the developer adds their malicious package into
>> the project, backdooring the codebase.

#### Mitigations

* Ensure all merge requests require review and approval from other engineers.
* Create administrative policy that mandates packages must only come from company approved
  sources
* Build and integrate tooling into CI pipelines as a technical control that reviews package files and
  identifies non-company sanctioned repositories; enforcing company policy
* Enable and use threat detection such as [GuardDuty](https://aws.amazon.com/guardduty/) to detect anomolies in patterns of service operation

#### Threat

>> A nation-state attacker proceeds to get one of their operatives employed with
>> the company. After performing reconnaissance on the corporate network, they
>> determine that the version control system has been operating with unpatched
>> vulnerabilities. The attacker proceeds to compromise the version control
>> server and implant malware which backdoors the company code releases

#### Mitigations

* Ensure corporate patch management policy exists that outlines SLA's for patching of
  systems
* Automate vulnerability scanning of systems and software to identify unpatched vulnerabilities
* Ensure the company performs background checks on all new employees
* Ensure the principal of least privilege is followed and employees who don't
  need access to systems are not provided access to systems
* Ensure the network is segregated so employees only have network access to the
  systems that are applicable to their job function
* Enable and use threat detection such as [GuardDuty](https://aws.amazon.com/guardduty/) to detect anomolies in patterns of service operation
* Ensure version control system is only available on the corporate VPN
* Ensure security groups / firewalls are configured to only allow necessary
  network access
* Ensure company hardening standards exist which outline procedures for
  hardening systems
* Ensure the version control system is hardened in alignment with corporate
  standards including things such as:
  * Removing all un-necessary services
  * Disabling root login
  * Disabling password login
  * Using a "golden image"
  * Using minimal containers if running a containerized environment
  * Etc.
* Implement a SIEM and forward events, syslog, etc.

#### Threat

>> An popular opensource dependency used by application gets compromised by an attacker.
>> The attacker implants bitcoin mining software into the dependency so that
>> consuming applications will mine bitcoin for the attacker

#### Mitigations

* Ensure automated dependency analysis is being performed on application dependencies to
  detect dependencies known to have been compromised
* Ensure lock files are being used so that builds repeatedly use the same
  version of a dependency until it is explicitly upgraded
* Enable and use threat detection such as [GuardDuty](https://aws.amazon.com/guardduty/) to detect anomolies in patterns of service operation
* Create administrative policy that mandates packages must only come from company approved
  sources
* Build and integrate tooling into CI pipelines as a technical control that reviews package files and
  identifies non-company sanctioned repositories; enforcing company policy
* Subscribe the engineering team to security disclosure distro lists so that

#### Threat

>> Developers are given access to the container registry so that they may
>> publish POC / testing images or pull production images for debugging
>> purposes. Container registry permissions are not well segmented and allow a
>> developer to push images to the same registry that production images are
>> pulled from. A developer proceeds to publish a backdoored image directly to
>> the repository and tags as latest, knowing that it will get picked up as
>> part of the next deploy.

#### Mitigations

* Ensure logging is configured for the container registry so that all image
  pushes and pulls are captured
* Do not use the same container registry for developer testing / POC that is
  used for production image storage
* Restrict publish access to the production container registry to only the CI/CD
  system
* Enable and use threat detection such as [GuardDuty](https://aws.amazon.com/guardduty/) to detect anomolies in patterns of service operation
* Implement image signing that will only allow the deployment of images that
  have been properly signed
* Review Google's [Binary Autnorization for
  Borg](https://cloud.google.com/security/binary-authorization-for-borg) and
    consider adopting some of their controls


#### What Else?

At this point I'm going to stop as this post is already getting kind of long but
I may come back and update with any more threats / mitigations that I feel may
be important and may be useful to others thinking about the threat model of a CI
pipeline. Feel free to leave comments on any you might think of to share with
others.


## What Should We Do About It?

At this point a team would review the threats and select any mitigations that
they felt should be implemented to address the identified threats. Since we
aren't building this for real, we'll obviously skip this step.

## Did We Do A Good Job?

Well... I suppose I'll leave that up to the readers to decide. There are
certainly more threats that I haven't identified here. This was only conducted
at 1 level of zoom. You could for example zoom in specifically on the build,
test, deploy multi-process circle of the diagram and see what threats you might
find there. Or perhaps, there is another diagram that would depict how an image
gets pulled from the registry and actually deployed to ECS / EKS / Fargate, etc.

I hope you have enjoyed seeing how I approach threat modeling and perhaps you
might feel encouraged to start doing something similar in your organization. Or
perhaps maybe some threats were identified here that maybe you hadn't considered
before and you might be inspired to review some of the security controls in your
organization.
