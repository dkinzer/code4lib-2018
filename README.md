How to set up a build projec QA site on PR events using Traefik.
====

## Abstract

Automatically creating a version of an application based on
submitted pull request (PR) code and adding a link from the PR to the build can
facilitate quicker turn around on feature iterations.  This post describes one
way to achieve this feature in your DevOps process by leveraging a new
reverse-proxy server called Traefik, that can be configured to serve
a Dockerized version of your application.

### What is Traefik
Traefik is a new powerful reverse proxy engine written in Golang.  It can
support all the traditional back-ends as well as newer ones and can do this
dynamically. That is the major selling point for Traefik and it's a big one.

Traefik is currently about 85% as fast a Nginx (as reported on the Traefik site
benchmark page). Considering how new this project is, and how much it can do,
that performance is rather impressive.

### Why use Traefik to set up QA sites
Although Traefik is a new comer to the reverse proxy world, it has some
relevant and exciting features that make it appropriate for our use case.

Now imagine if you were trying to set up a service to dynamically create a QA
sites for your project:

Well, Traefik can be used to proxy to docker containers and can do this
dynamically.  This is huge because our use case will be to create QA sites for
PRs using docker containers.  Traefik can also be used on non container
applications, including web or even flat files.


## Plan
Conceptionally the requirements of this effort are easy to grok once we've
settled on using Traefik with the docker back-end:

* First the web application we want to stage PRs for needs to be Dockerized
  (that's sort of a given really).
* Second, we need a way to interact dynamically with GitHub on events like PRs
  created and updated and merged, etc..  We can leverage the well known
  application Jenkins CI for this task since it's a stable project with plenty
  of GitHub web-hook plugins to choose from.
* Third we will need to do things like  download a copy of the PR repo,
  checkout PR branch or pull latest changes on branch, run `docker-compose up
  -d` from within the PR repo source directory etc. We could of course do this
  inside of a Jenkins' job shell script, but that can get unwieldy to maintain,
  so instead I've encapsulated this concern into a gem created named Roper that
  I will use from withing the Jenkins job.


## Execution

### Set up Traefik
There are several options for setting up Traefik, including installing the
binary.  But, since we will be using Docker to deploy our apps anyway, it makes
sense to also use Docker to deploy Traefik. A quick search on GitHub should
lead you to a Dockerized setup for Traefik using the official Traefik docker
image.

And that is precisely what I did. I found a project that sets up Traefik to
proxy a Dockerized back-end. I made some minor changes to the config file
`Traefik.toml` to fit my needs and spun it up:

```
git clone git@github.com:dkinzer/reverse-proxy.git
cd reverse-proxy.git
docker-compose up
```
And with that I have Traefik running as a service, and waiting for
a docker-image to be spun up to serve on port 443.

### Configuring your Dockerized app to work with Traefik.
Although Dockerizing your application is outside the scope of this post, there
are some Traefik specific points that need to be kept in mind.

It's important to note that Traefik needs to belong to the same network as the
back-end that it is serving. And if the back-end belongs to multiple networks
then it needs to be configured so that it always communicates with Traefik
using the network that they both share.  This seems obvious in retrospect but
I spent many hours trying to debug a situation where Traefik was randomly
picking one network over another.

When using docker-compose to spin up your app, the way to configure what
network Traefik listens to is via labels that can be passed configured.

In fact using these same labels you can configure whether or not Traefik will
serve the Dockerized image at all (important when your app is composed of
multiple services with one public facing service).

Below is an example docker-compose file that has been set up to work with
Traefik:

```yaml
services:
  app:
    build:
      context: .
      dockerfile: .docker/app/Dockerfile
    container_name: "$ROPER_REPO_BRANCH"
    ports:
      - "3000"
    environment:
      SOLR_URL: "http://solr:8983/solr/blacklight"
      LC_ALL: "C.UTF-8"
    labels:
      - "traefik.enable=true"
      - "traefik.frontend.rule=Host:${ROPER_REPO_BRANCH}.${DOMAIN}"
      - "traefik.backend=microbot"
      - "traefik.docker.network=reverseproxy_default"
    networks:
      - "reverseproxy_default"
    depends_on:
      - solr
    entrypoint:
      - ruby
      - bin/app-start.rb
```
Note that in the section above we can set up even the front-end rules for what
the service Host will be.  We'll talk more about the environment variables
being set in that line in the section below on the Roper gem.

You can learn more about what labels are available for configuring a Docker
back-end at: https://docs.traefik.io/configuration/backends/docker/

### Configuring the Jenkins Web-hook Job
Since the goal is to get something to happen when a GitHub PR is generated,
then we can use a GitHub web-hook to post a Jenkins Job:
https://developer.github.com/webhooks/. The Jenkins Job in turn could trigger
the setting up of containerized application.

For our particular case we decided to use two Jenkins Jobs.  One job is to
handle the web-hook and the other one is a parameterized job to actually
trigger the build.  The reason for this two step process is that with the
parameterized build we can run rebuilds in the cases where a build goes wrong
(or just because), and that is a nice feature to have.

For the sake of brevity we'll paraphrase each of the jobs below with links to
the full job definitions:

#### Trigger PR QA Build
This job leverages the Jenkins Generic Trigger plugin. It uses JSONPaths to
parse the GitHub web-hook payload:

| key   | JSONPath  |
|------|:------:|
| repo | $.pull_request.head.repo.full_name |
| branch | $.pull_request.head.ref |
| sha | $.pull_request.head.sha |
| action | $.action |

We then define the following filter in order to only trigger on specific
matches.

| filter | match |
| ----| :---:|
| `$X_GitHub_Event $repo $action` | `pull_request tulibraries\/tul_cob (open\|close\|synchronize)` |

Thus we limit the build triggers to pull requests coming from PRs generated
from a specific organization and repository.

For the build step we use the parameters we collected to trigger a parameterized job.

#### Parameterized Roper Builder
The receiving job is a parameterized job that simply calls out to the actual building process.

```bash
args="--repo=$repo --branch=$branch"

if [ "$action" == "closed" ]; then
  command=release
else
  command=lasso
  args=";$args --domain=devkinzer.net --sha=$sha --status_url=${BUILD_URL}console";
fi

ssh devkinzer -C roper $command $args
```

So as you can see we are calling out to an executable named `roper` and
specifying arguments for a build.


### Configuring the GitHub Web-hook.
Once we have created a Jenkins Job that we can trigger via a web-hook post then
its time to configure the GitHub web-hook.  There are instructions on GitHub of
how to go about this so I wont go into any details here.  The important bit for
us is to before sure that:

* The web-hook is set to have content type application/json
* And, it should trigger at least for PRs and push events.


### Roper
Of course instead of calling out to a build executable we could have written a
build script directly into a Jenkins job.  Such a script would take care of all the
steps for properly launching the app such as:

* Posting to the GitHub PR a status of change
* Cloning the PR and checking out the correct reference
* Running `docker-compose up` within the repository's working directory.
* Posting a link to the built image that gets served by Traefik
* Handling updates to the PR properly
* Handling PR closing such as removing the images and files used in the build.

Although this list is small an relatively straight forward.  Such a script
could easily become unwieldy to maintain as a Jenkins Job shell step. Instead I
chose to wrap this functionality into a gem which I named Roper. As a gem I can
apply tests to all of the features and separate out concerns to make this
easier to understand user and maintain in the long run.

In addition to the steps above Roper adds some environment variables that are
available to docker-compose. Thus in the example above:

```
      - "traefik.frontend.rule=Host:${ROPER_REPO_BRANCH}.${DOMAIN}"
```

`ROPER_REPO_BRANCH` is made available by Roper. For more details on Roper see
the [Roper documentation](http://www.rubydoc.info/github/tulibraries/roper/master)

