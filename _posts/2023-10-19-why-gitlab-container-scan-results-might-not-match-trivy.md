---
layout: post
title: "Why GitLab Container Scan Results Might Not Match Trivy"
description: 'Sometimes you may see different vulnerabilities when Trivy is run locally vs when GitLab container scan runs. Here are some reasons why.'
tags: [gitlab, vulnerability management, trivy, container scanning]
---

As a security engineer you often need to interpret scan results. If you work with
GitLab container scanning enough, you may encounter situations where the vulnerabilities that GitLab reports don't match what is found when the same container image is
scanned with Trivy locally. Here are some reasons why that might occur.

## CS\_IGNORE\_UNFIXED

One reason that this might occur is that the `CS_IGNORE_UNFIXED` environment
variable may be configured somewhere. Possibly at the project, group, or global
level as a CI/CD variable or possibly in a template file. [As described in
GitLab's documentation](https://docs.gitlab.com/ee/user/application_security/container_scanning/), this variable "Ignores vulnerabilities that are not fixed." This can be a useful variable to have set depending on an organizations vulnerability management processes and the type of images being scanned. (Some Debian / Ubuntu based images can have many "vulnerabilities" that their security teams have assessed to be of minor importance and won't fix, which ultimately add noise to a scan report.) However, when Trivy is run locally, if the `--ignore-unfixed` flag is not specified the unfixed vulnerabilities that GitLab scanning has been hiding will appear.

## GitLab Enterprise Database

Another reason that GitLab container scanning may report different results than
a local Trivy scan is due to the vulnerability database that is used. Even
though GitLab container scanning uses Trivy, it uses GitLab's own database.
Additionally, depending on if GitLab Enterprise Edition is running the scan vs.
GitLab Community Edition, different vulnerability databases are used. We can see
where the databases are installed in the scanner source code [here](https://gitlab.com/gitlab-org/security-products/analyzers/container-scanning/-/blob/master/script/setup.sh?ref_type=heads#L23-29):

```bash
echo "Dowloading CE Trivy DB"
mkdir -p /tmp/trivy-ce
oras pull "$CE_TRIVY_DB_REGISTRY":"${trivy_db_version_ce}" -a -o /tmp/trivy-ce

echo "Dowloading EE Trivy DB"
mkdir -p /tmp/trivy-ee
oras pull "$EE_TRIVY_DB_REGISTRY":"${trivy_db_version_ee}" -a -o /tmp/trivy-ee
```

And then [depending on which environment the scanner is running in, which vulnerability database is used](https://gitlab.com/gitlab-org/security-products/analyzers/container-scanning/-/blob/master/lib/gcs/trivy.rb?ref_type=heads#L131-136).


```ruby
def cache_dir
  if Gcs::Environment.ee?
    File.join(CACHE_DIR_BASE, "ee")
  else
    File.join(CACHE_DIR_BASE, "ce")
  end
end
```


Trivy uses the [GitHub Advisory Database](https://github.com/advisories) so if a vulnerability is in one database and not another, or if the vulnerability signature definition differs between one database and the other, different results may be encountered.

Something else worth noting is that because GitLab uses a different database for
enterprise edition and community edition, different results can be encountered
when running the GitLab container scanner locally vs. when it runs in a
pipeline.

## CS\_DISABLE\_LANGUAGE\_VULNERABILITY\_SCAN

The GitLab container scanning configuration variable of `CS_DISABLE_LANGUAGE_VULNERABILITY_SCAN` disables the programming language dependency analysis of the container scanner and defaults to true in GitLab. This is because GitLab has a separate dependency scanner. However, a container image may have other application code installed in them that the dependency scanning in GitLab wouldn't find but Trivy would. If you are seeing application dependency type vulnerabilities in a local Trivy scan but not in GitLab, this could be the culprit.

_Note: Running Trivy with the `-f json` flag can be extremely useful when Trivy reports an application level vulnerability in a container image because it can point you to the file path where the code is installed. If this flag doesn't report useful results in tracing down where Trivy is reporting the vulnerability, try upgrading Trivy as this reporting has been improved in more recent versions of Trivy. Additionally, depending on your Trivy version a vulnerability may be detected in a jar or war but you may be left wondering where within it the vulnerable dependency is. This may occur due to ["shading" or "shadowing"](https://softwareengineering.stackexchange.com/questions/297276/what-is-a-shaded-java-dependency) of dependencies, where a dependency is essentially vendored into the jar/war. In this situation you may need to unzip the jar / war and do some grepping to find what Trivy is flagging. Recent versions of Trivy are better at pointing out the shaded dependency._

## The Trivy Version

Not as likely but always worth checking is the Trivy version. Trivy has added
support for application level vulnerability scanning, secret scanning, etc. and
is improving all the time so it could just be that the version of Trivy that is
running locally may not align with what GitLab is running. Checking the versions
and ensuring the same scanner is being run locally so there is as much of an
apples to apples comparison can sometimes be useful.

## Conclusion

Hopefully this post will save some people some head scratching when container
scanning results don't quite match up with what GitLab and a local Trivy scan
report. Armed with the knowledge above I think most scanner discrepencies will be covered. There are likely more things I haven't encountered before so if you know of any more gotchas like these please leave them in the comments.
