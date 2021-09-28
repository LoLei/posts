# How to Fix Empty Project in SonarQube Docker

This post shows one way to fix an issue that may cause a SonarQube project to show up as empty only
when the scan is run in a container environment.

## TL;DR
Concerning the issue: **"This project is empty"** in the SonarQube UI:  
SonarQube does not like being run as the root user. Specify a different user to be used in the
container, and run the scan from this user's home directory.

A containerfile like this suffices:

```Dockerfile
FROM docker.io/openjdk:11
RUN useradd --create-home --shell /bin/bash sonarqube
USER sonarqube
WORKDIR /home/sonarqube
```

## More Thorough Explanation

I've been setting up a few projects to be analyzed by [SonarQube](https://www.sonarqube.org/)
recently. A TypeScript project had its own issues that may be worth for a blog post, but in this
Kotlin project this post is about I didn't find any solutions online, so **I'm making this post
solely in hopes of shortening the journey for people who stumble upon the same issue.** It isn't
super interesting, so if you're not experiencing this issue yourself you may as well stop reading.

In this Kotlin Gradle project, I noticed that the project shows up as an empty project in the
SonarQube UI, however only if the scan was triggered from within a container environment. I've used
both the SonarScanner Gradle plugin and the CLI, both of which worked fine locally, but when using
it from within a container it presented an empty project as a result. This led me on a wild goose
chase of trying to find out the differences between the locally triggered scan and the one that's
started via CI / a container. (After even realizing that it's not just the CI's fault, but the
container environment's.) In the end, it turns out that SonarQube cannot handle not being run from a
"normal" user environment. Most images use the root user by default, which does not count as such.

What makes it work is creating a non-root user and switching to that user's home directory before
executing the scan, of course with the code to be scanned in that directory, or a subdirectory, as
well. For CI purposes, that can be a base image as listed in the TL;DR above (if it's not a Java
project the `FROM` directive may differ naturally), or even one that just includes a line such as
this:

```Dockerfile
RUN useradd --create-home --uid 1000 --shell /bin/bash sonarqube
```

As long as the user and the home directory exists, and the CI provider facilities overwriting the
default user with whom the pipeline steps are run, this is enough. On Bitbucket Pipelines, this can
be done with the `run-as-user` step config option, which needs a UID of an existing user in the
image. Since the pipeline may also clone the repo into some other directory, its contents may also
need to be copied to the desired user's home directory.

## References
The only marginally related discussion about this I could find online is
[this](https://community.sonarsource.com/t/sonar-plugin-for-gradle-doesnt-work-when-run-inside-openjdk-11-12-docker-container/9622/2)
one, which has Java user woes as well, but triggers a different kind of bug. For completeness' sake,
many other potential solutions for the "This project is empty" problem can be found
[here](https://community.sonarsource.com/t/this-project-is-empty/7961/4).

## Addendum
* I hope my SEO is good enough so search engines will display this post for people with relevant
  interest.
* I mentioned "Docker" in the title for this purpose, but it should be the same for any container
  engine/runtime. I tested it with Podman as well.

