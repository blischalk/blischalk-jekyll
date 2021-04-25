---
layout: post
title: "Amazon ECR Image Scanning Gotchas"
description: 'Things you should know about the "vulnerabilities" ECR image scan results report'
tags: [docker, aws, ecr, clair, vulnerabilities, compliance, scanning]
---

After working with ECR image scanning for a while now I felt that there are
some lessons learned that would be worth sharing for others who may use the tool. Here are a couple gotchas I've encountered that you may be interested to know about.

## Kernel Vulnerability False Positives

As docker images don't contain a linux kernel and use the host system kernel,
you may be surprised to find that Amazon ECR image scanning will report kernel vulnerabilities for your images even though images don't have an
OS kernel. Why might this be? After investigating the issue I learned that under
the hood, Amazon image scanning [uses the Clair vulnerability scanner](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html) - [Clair](https://github.com/quay/clair) to scan images for vulnerabilities. And after further research I found [this](https://github.com/quay/clair/blob/master/Documentation/running-clair.md):

>I'm seeing Linux kernel vulnerabilities in my image, that doesn't make any sense since containers share the host kernel!

> Many container base images using Linux distributions as a foundation will install dummy kernel packages that do nothing but satisfy their package manager's dependency requirements. The Clair developers have taken the stance that Clair should not filter results, providing the most accurate data as possible to user interfaces that can then apply filters that make sense for their users.

Two major problems that this causes in Amazons Clair implementation is that:

1. AWS ECR image scanning doesn't provide any mechanism to triage vulnerabilities and maintain state. This means that there is no way to mark the kernel issues as false positives removing them from the current and/or subsequent scans.
2. If these scan results are being used for compliance audits, you may find
   yourself explaining to management and auditors how containerization works
  and to pay no attention to the man behind the curtain because kernel
  vulnerabilities don't impact images; linux distros include dummy
  packages; and the Clair maintainers feel etc. etc.

## Vendor Dependant Vulnerabilities / False Positives

This problem is not unique to Clair or Amazon's implementation of it, but
something that you will definitely encounter when you start doing image
vulnerability scans. There are many "vulnerabilities" that image scanning tools
like Clair will report that are actually contested by the linux distro's
security team and have indicated that the distro is not vulnerable. As an
example, even though a scan may indicate that Nginx version 1234 may have CVE-xxx-xxxx, because the
distro doesn't compile that package with a particular compiler flag set, then
the distro's security team considers the finding to be a false positive or
unimportant.

Obviously it is not possible for scanners to know all of the conversations that take place in the various linux distro's bug trackers and how one distro may be compiling a package different than another. The tool is just doing the best it can. However, this is something to be aware of and going along with the points above:

1. Since there is no way to triage issues in AWS ECR image scanning, this adds a
lot of noise to sift through. Keeping track of what you have researched
vs. what you haven't also becomes more difficult.
2. If these results are being used for compliance audits, again be prepared to
   have a laundry list of explanations along the lines of, "the security team of
   distro x has indicated that their distro doesn't compile the package with the
   flag that would make the package vulnerable and so this is a false
   positive, etc. etc."

## Lessons Learned

I learned quite a bit about the current state of image scanning during my
experience with Amazon ECR image scanning. I would love for
Amazon to build out ther image scanning functionality a bit more to allow for
triaging vulnerabilities the way you might in a tool like BlackDuck or Coverity.
If vulnerability reporting for compliance obligations is important to you though, selecting
another tool that provides more features / configurability like
[Trivy](https://github.com/aquasecurity/trivy) may be a better solution for your
image scanning needs.
