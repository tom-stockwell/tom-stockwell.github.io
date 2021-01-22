---
title: "Migrating Applications to OpenShift, Part 2: Proof of Concept"
draft: true
description: "In this part of the Migrating Applications to OpenShift series, I will demonstrate creating a proof of concept application using bookinfo."
---

In this part of the [Migrating Applications to OpenShift](1-overview.md) series, I will demonstrate creating a proof of concept application using bookinfo.

I will showcase Kubernetes (k8s) YAML changes using [oc patch](https://docs.openshift.com/container-platform/3.11/cli_reference/basic_cli_operations.html#patch), but it is possible to make the same changes by directly modifying the Pod YAML via [oc edit](https://docs.openshift.com/container-platform/3.11/cli_reference/basic_cli_operations.html#edit) or through the GUI.

## Getting Started

First up we make sure to have a bookinfo project setup on OpenShift and check out the bookinfo repositories from GitHub.

```shell
oc login
oc new-project bookinfo
mkdir bookinfo
cd bookinfo
git clone https://github.com/stocky37/bookinfo-mongodb.git mongodb
git clone https://github.com/stocky37/bookinfo-ratings.git ratings
git clone https://github.com/stocky37/bookinfo-details.git details
git clone https://github.com/stocky37/bookinfo-reviews.git reviews
git clone https://github.com/stocky37/bookinfo-mongodb.git productpage
```

This will checkout develop branch by default, commands lower down will make sure to deploy the app from master branch code if necessary.

### Useful Commands

Unless otherwise specified, I use the following commands regularly to check the status and progress of builds & deployments:

- **`oc status`** - Gives a general overview of the entire project.
- **`oc get pods`** - Shows the name & status of all Pods in the project.
- **`oc logs -f bc/<app>`** - Follows the logs of the current build Pod for `<app>`.
- **`oc logs -f dc/<app>`** - Follows the logs of the deployment/deployed Pod for `<app>`.
  Until the application Pod deploys, it will show the deploy Pod logs.
  After the deploy Pod completes and exits, the command will complete, after which you can rerun the command to follow the application logs.

## Ratings Database (MongoDB)

For no reason in particular, I've selected MongoDB as the backend for the Ratings Service over PostgreSQL (the Ratings Service can use either) and will deploy this first so that the other services have a database they can use.

```shell
cd mongodb
git checkout openshift3
```

The source of the istio image can be found at [`Dockerfile`](https://github.com/stocky37/bookinfo-mongodb/blob/master/Dockerfile).
Upon investigation, we can see that the Dockerfile extends an existing MongoDB image and pre-populates the database with some data.
We can implement this more concisely in OpenShift without needing to create a custom image.

Firstly, we should create a MongoDB instance using the `mongodb-persistent` template.
This template will instantiate all k8s resources, including persistent storage, to run a persistent MongoDB database in OpenShift.

```shell
oc new-app --template mongodb-persistent --name mongodb
```

Once the MongoDB pod is up and running, we can create a [remote shell](https://docs.openshift.com/container-platform/3.11/dev_guide/ssh_environment.html) and connect to the local database.

```shell
oc rsh dc/mongodb bash -c 'mongo -u $MONGODB_USER -p $MONGODB_PASSWORD $MONGODB_DATABASE --quiet --eval "db.getCollectionNames()"'
```

Should display an empty array (`[ ]`), since no collections should exist with an empty database.

> **Note:** The environment variables I used to connect to MongoDB above are sourced from the `mongodb` secret and assigned to the Pod by `oc new-app`.

Now that we have a functioning MongoDB database, we need to ensure it is pre-populated with the data listed in [`ratings_data.json`](https://github.com/stocky37/bookinfo-mongodb/blob/master/ratings_data.json).
Using a method mentioned in [Using Post Hook to Initialize a Database](https://www.openshift.com/blog/using-post-hook-to-initialize-a-database), we can reuse the scripts the Dockerfile used (with some small modifications) to populate our database after the Pod starts by using a [Pod-based lifecycle hook](https://docs.openshift.com/container-platform/3.11/dev_guide/deployments/deployment_strategies.html#pod-based-lifecycle-hook).

We are going to have to make the following changes to [`script.sh`](https://github.com/stocky37/bookinfo-mongodb/blob/master/script.sh) for it to work in our lifecycle hook:

- Enable the MongoDB Red Hat Software Collection to access the MongoDB binaries
- Connect to our MongoDB instance using the appropriate environment variables
- Handle duplicate values on import using [`--upsertFields`](https://docs.mongodb.com/manual/reference/program/mongoimport/#cmdoption-mongoimport-upsertfields) since our script will be run after every deployment.

View the updated script [here](https://github.com/stocky37/bookinfo/blob/migrating-apps-v3/1.0/script.sh).

Now we need to get the script and data mounted into the mongodb Pod so that we can run them when the Pod starts.
The best way to do this is using a [`ConfigMap`](https://docs.openshift.com/container-platform/3.11/dev_guide/configmaps.html).

```shell
oc create cm mongodb-scripts --from-file=script.sh --from-file=ratings_data.json
```

Now that we've created the ConfigMap, we can mount it into the Pod as a volume and set up a Post Lifecycle Hook that uses the script.

View the patch [here](https://github.com/stocky37/bookinfo-mongodb/blob/migrating-apps-v3/1.0/.k8s/patches/1-dc-hook.yml).

```shell
oc patch dc mongodb -p "$(cat k8s/patches/1-dc-hook.yml)"
```

> **Note:** The extra `sleep` command in the lifecycle hook command gives the MongoDB service a chance to start up successfully before the script runs.

After the rollout succeeds, we can query the MongoDB `ratings` collection to check that it has been populated with the data in [`ratings_data.json`](https://github.com/stocky37/bookinfo/blob/master/.k8s/patches/1-dc-hook.yml).

```shell
oc rsh dc/mongodb bash -c 'mongo -u $MONGODB_USER -p $MONGODB_PASSWORD $MONGODB_DATABASE --quiet --eval "db.ratings.find()"'
```

We should see something like the following:

```json
{ "_id" : ObjectId("..."), "rating" : 5 }
{ "_id" : ObjectId("..."), "rating": 4 }
```

And there it is, a pre-populated MongoDB database running on OpenShift!

## Ratings Service (NodeJS)

```shell
cd ../ratings
git checkout openshift3
```

- Initial build from `master` with nodejs s2i

```shell
oc new-app 'nodejs~https://github.com/stocky37/bookinfo-ratings.git#master' --name ratings
--> Found image 0d01232 (7 months old) in image stream "openshift/nodejs" under tag "10" for "nodejs"
...
--> Success
```

```shell
oc status
...
svc/ratings - x.x.x.x:8080
  dc/ratings deploys istag/ratings:latest <-
    bc/ratings source builds https://github.com/stocky37/bookinfo-ratings.git on openshift/nodejs:10
    deployment #1 deployed 2 minutes ago - 0/1 pods (warning: 2 restarts)

Errors:
  * pod/ratings-1-5t966 is crash-looping
...
```

- Doesn't start - crash looping on deployment
- Should check the deployment logs

```shell
oc logs -f dc/ratings
...
> @ start /opt/app-root/src
> node ratings.js

net.js:1405
      throw new ERR_SOCKET_BAD_PORT(options.port);
      ^

RangeError [ERR_SOCKET_BAD_PORT]: Port should be >= 0 and < 65536. Received NaN.
...
```

- Looks like the script expects a port to be passed to it on the cmd line

```js
var port = parseInt(process.argv[2]);
```

- According to the [NodeJS S2I Readme](https://github.com/sclorg/s2i-nodejs-container/tree/master/10#environment-variables), we can use the `NPM_RUN` environment variable to override the script that gets run when the container starts.
- In this case, we can set it to the following to set a port: `start -- 8080`. `start` ensures it still runs the `start` script, the `--` indicates everything after it should be an argument to the underlying command run by the previous script, and in our case we want to give it the port number `8080` as it is the default port expected by services in k8s/openshift?.
- Two ways to provide envvars, through `.s2i/environment`, and through the `deploymentconfig`.
- Should use `.s2i/environment` for something that will not change for the app in different environments (dev, test, etc.) and `deploymentconfig` for others
- In this case we have the first one so should do that

<!-- TODO: add link to .s2i/environment -->

- View it here.

- Now we'll patch the buildconfig to point to the updated commit

<!-- TODO: add link to patch -->

- View the patch.

```shell
oc patch bc ratings -p "$(cat .k8s/patches/1-bc-ref.yml)"
buildconfig.build.openshift.io/ratings patched
```

- Patched the build, so need to make sure we start a new build (may happen automatically?)

```shell
oc start-build ratings
build.build.openshift.io/ratings-2 started
```

- This will eventually work

```shell
oc status
...
svc/ratings - x.x.x.x:8080
  dc/ratings deploys istag/ratings:latest <-
    bc/ratings source builds https://github.com/stocky37/bookinfo-ratings.git#migratings-apps-v3/1.0 on openshift/nodejs:10
    deployment #2 deployed about a minute ago - 1 pod
    deployment #1 deployed 3 hours ago
...
```

- Appears to be running, check by exposing service and hitting up an endpoint

```shell
oc expose svc ratings
route.route.openshift.io/ratings exposed

host="$(oc get route ratings --template '{{.spec.host}}')"

curl "http://$host/ratings/1"
{"id":1,"ratings":{"Reviewer1":5,"Reviewer2":4}}
```

- Yay! However, we're not actually hooked up to the database just yet. To do so we need to make sure the `SERVICE_VERSION` environment variable is set to `v2` and set the `MONGO_DB_URL` environment variable to point to our mongodb service.

- Due to the default [DNS-based service discovery](https://docs.openshift.com/container-platform/3.11/architecture/networking/networking.html#architecture-additional-concepts-openshift-dns) for services, can just use `mongodb:27017`.

```shell
oc patch dc ratings -p "$(cat .k8s/patches/2-dc-mongo.yml)"
deploymentconfig.apps.openshift.io/ratings patched
```

- Once that's done, check the api again

```shell
curl "http://$host/ratings/1"
{"error":"could not connect to ratings database"}
```

- This will still fail. The default mongodb on openshift is secured so we need to modify the application to work with a secured cluster (**to do:** how do we figure this out, show debug steps). I made the following change (maybe link to diff instead? | could I import with asciidoc?):

```javascript
var MongoClient = require("mongodb").MongoClient;
var host = process.env.MONGO_DB_URL;
var database = process.env.MONGODB_DATABASE;
var username = process.env.MONGODB_USER;
var password = process.env.MONGODB_PASSWORD;
var url = `mongodb://${username}:${password}@${host}/${database}`;
```

- Now, update the deployment to add the new envvars we defined, and update the build to point to the updated code

```shell
oc patch dc ratings -p "$(cat.k8s/patches/3-dc-mongo.yml)"
deploymentconfig.apps.openshift.io/ratings patched

oc patch bc ratings -p "$(cat.k8s/patches/4-bc-ref.yml)"
buildconfig.build.openshift.io/ratings patched

oc start-build ratings
build.build.openshift.io/ratings-3 started
```

- Check it worked

```shell
oc status
...
http://ratings-bookinfo.apps.cluster.example.com to pod port 8080-tcp (svc/ratings)
  dc/ratings deploys istag/ratings:latest <-
    bc/ratings source builds https://github.com/stocky37/bookinfo-ratings.git#migrating-apps-v3/1.0 on openshift/nodejs:10
    deployment #4 deployed about a minute ago - 1 pod
    deployment #3 deployed 7 minutes ago
    deployment #2 deployed about an hour ago
...

curl "http://$host/ratings/1"
{"error":"could not connect to ratings database"}
```

- Note that in the real world we'd probably just make an update and push our code to master.
  I've just used tags here to demostrate all the different changes for this blog.

## Details Service (Ruby)

```shell
cd ../details
git checkout openshift3
```

- Start with initial deploy to OpenShift

```shell
oc new-app 'ruby~https://github.com/stocky37/bookinfo-details.git#master' --name details
--> Found image 18a91a0 (11 months old) in image stream "openshift/ruby" under tag "2.5" for "ruby"
...
--> Success

oc status
svc/details - x.x.x.x:8080
  dc/details deploys istag/details:latest <-
    bc/details source builds https://github.com/stocky37/bookinfo-details.git#master on openshift/ruby:2.5
    deployment #1 deployed 2 minutes ago - 0/1 pods (warning: 4 restarts)
...
Errors:
  * pod/details-1-w7cqw is crash-looping
```

- Appears to have failed, let's find out why

```shell
oc logs dc/details
You might consider adding 'puma' into your Gemfile.
ERROR: Rubygem Rack is not installed in the present image.
       Add rack to your Gemfile in order to start the web server.
```

- Odd. Looks like our app doesn't conform to the defaults required by the ruby s2i image.
- Tried looking for doco here: https://github.com/sclorg/s2i-ruby-container/tree/master/2.5, didn't find anything useful
- Check the source code instead
- https://github.com/sclorg/s2i-ruby-container/blob/master/2.5/s2i/bin/run
- Looks like it's expected to be a rack app, but ours is not
- To fix this, let's completely override the default ruby run script with a script that will run our app the way we need
- https://docs.openshift.com/container-platform/3.11/using_images/s2i_images/customizing_s2i_images.html
- View the .s2i/bin/run script: todo: add link
- Update the git ref to point to one I've already completed

```shell
oc patch bc details -p "$(cat .k8s/patches/1-bc-ref.yml)"
buildconfig.build.openshift.io/details patched

oc start-build details
build.build.openshift.io/details-2 started

oc status
svc/details - x.x.x.x:8080
  dc/details deploys istag/details:latest <-
    bc/details source builds https://github.com/stocky37/bookinfo-details.git#migratings-apps-v3/1.0 on openshift/ruby:2.5
    deployment #2 running for 28 seconds - 1 pod
    deployment #1 deployed 18 minutes ago
...
```

- Looks like it worked, let's check the api

```shell
oc expose svc details
route.route.openshift.io/details exposed

host="$(oc get route details --template '{{.spec.host}}')"
curl "http://$host/details/1"
{"id":1,"author":"William Shakespeare","year":1595,"type":"paperback","pages":200,"publisher":"PublisherA","language":"English","ISBN-10":"1234567890","ISBN-13":"123-1234567890"}
```

## Reviews Service (Java + Gradle)

```shell
cd ../reviews
git checkout openshift3
```

- Have a JEE Application, needs to run in an JEE server container - e.g. wildfly
- if it doesn't exist (use `oc get is -n openshift | grep wildfly` to check), either ask your ocp admin to add it (https://github.com/wildfly/wildfly-s2i) or create the imagestreams locally in your project yourself

```shell
cd $(mktemp -d)
git clone https://github.com/wildfly/wildfly-s2i.git
cd wildfly-s2i
oc create -f templates
imagestream.image.openshift.io/wildfly created
imagestream.image.openshift.io/wildfly-runtime created
template.template.openshift.io/wildfly-s2i-chained-build-template created
```

- Run oc new-app

```shell
oc new-app 'wildfly~https://github.com/stocky37/bookinfo-reviews.git#master' --name reviews
--> Found image bdf6490 (4 weeks old) in image stream "bookinfo/wildfly" under tag "latest" for "wildfly"
...
--> Success

oc status
...
svc/reviews - 172.30.88.245 ports 8080, 8778
  dc/reviews deploys istag/reviews:latest <-
    bc/reviews source builds https://github.com/stocky37/bookinfo-reviews.git#master on istag/wildfly:latest
    deployment #1 deployed 2 minutes ago - 1 pod
```

- Looks ok at first glance, let's check the api

```shell
oc expose svc reviews
route.route.openshift.io/reviews exposed

host="$(oc get route reviews --template '{{.spec.host}}')"
curl "http://$host/reviews/1"
<html><head><title>Error</title></head><body>404 - Not Found</body></html>
```

- Hmmm, that's not right, let's check the logs

```shell
oc logs dc/details
...

oc logs bc/details
...
INFO S2I source build with plain binaries detected
INFO Copying deployments from . to /deployments...
...
```

- Nothing jumps out in the `dc` logs - though there are signs something is amiss. There are no logs adding/starting the `war` file that should be built by the build
- In the `bc` logs however, something seems wrong. It seems to think it should be doing a binary s2i deployment, but we want it to build from source. What's going on?
- The s2i image doens't work with gradle, and assumed a binary build when it didn't find a `pom.xml`
- We know we can customise the `run` script of an s2i image, but we can also customise the `assemble` script
- Let's customise it to handle gradle
- First, however, we need to add the gradle wrapper to the code so we can run gradle from anywhere
- Follow the instructions at https://docs.gradle.org/current/userguide/gradle_wrapper.html
- Wanted to use the code from the s2i image script in my script, so tried to find it.
- It's just a yaml file though, https://github.com/wildfly/wildfly-s2i/blob/master/wildfly-builder-image/image.yaml, looks like it's using https://github.com/cekit/cekit
- According to README, https://github.com/wildfly/wildfly-cekit-modules holds most of the modules. We're after those related to s2i
- https://github.com/wildfly/wildfly-cekit-modules/blob/master/jboss/container/wildfly/s2i/bash/module.yaml looks to be the right one, it's installing `jboss.container.maven.s2i.bash`
  - is found in the other linked repo: https://github.com/jboss-openshift/cct_module/tree/master/jboss/container/maven/s2i
- Assemble script: https://github.com/jboss-openshift/cct_module/blob/master/jboss/container/maven/s2i/artifacts/usr/local/s2i/assemble
- Helper functions: https://github.com/jboss-openshift/cct_module/blob/master/jboss/container/maven/s2i/artifacts/opt/jboss/container/maven/s2i/maven-s2i
- Here is the new `assemble` script I developed (link to repo)
- It uses the most of the same functions as the original `assemble` script, but changes the build to a customiseable gradle build
- Notice it still uses the copy artifacts function, so now we have to update the `MAVEN_S2I_ARTIFACT_DIRS` to the output of our build at `reviews-application/build/libs`
- As before, this value will not change per deploy environment so we'll set it in `.s2i/environment`

- Update our ref to these changes

```shell
oc patch bc reviews -p "$(cat .k8s/patches/1-bc-ref.yml)"
buildconfig.build.openshift.io/reviews patched

oc start-build reviews
build.build.openshift.io/reviews-2 started
```

- Check the logs to make sure things worked

```shell
oc logs bc/reviews
...
Welcome to Gradle 6.1.1!
...
BUILD SUCCESSFUL in 41s
...

oc logs dc/reviews
...
14:32:14,560 INFO  [org.jboss.as.server.deployment] (MSC service thread 1-1) WFLYSRV0027: Starting deployment of "reviews-application-1.0.war" (runtime-name: "reviews-application-1.0.war")
...
14:32:30,751 INFO  [org.jboss.resteasy.resteasy_jaxrs.i18n] (ServerService Thread Pool -- 80) RESTEASY002225: Deploying javax.ws.rs.core.Application: class application.ReviewsApplication
...
14:32:30,968 INFO  [org.wildfly.extension.undertow] (ServerService Thread Pool -- 80) WFLYUT0021: Registered web context: '/reviews-application-1.0' for server 'default-server'
14:32:31,349 INFO  [org.jboss.as.server] (ServerService Thread Pool -- 46) WFLYSRV0010: Deployed "reviews-application-1.0.war" (runtime-name : "reviews-application-1.0.war")
...
```

- Looks like it worked. Though there may be one problem left due to , let's try hitting up the api

```shell
curl "http://$host/reviews/1"
<html><head><title>Error</title></head><body>404 - Not Found</body></html>
```

- Hmmm, still not quite there. Hint from `Registered web context: '/reviews-application-1.0'`, so let's try it:

```shell
curl "http://$host/reviews-application-1.0/reviews/1"
{"id": "1","reviews": [{  "reviewer": "Reviewer1",  "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!"},{  "reviewer": "Reviewer2",  "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare."}]}
```

- Progress! Now we just have to deploy our application in to the root context of the JEE server
- Easiest way to do that is to name the war file ROOT.war for when it is deployed (it is part of the JEE Servlet Spec) (link?)
- We'll do this by updating `reviews-application/build.gradle` - link to diff
- Now update the build git ref to use these changes as well

```shell
oc patch bc reviews -p "$(cat .k8s/patches/2-bc-ref.yml)"
buildconfig.build.openshift.io/reviews patched

oc start-build reviews
build.build.openshift.io/reviews-3 started

curl "http://$host/reviews/1"
{"id": "1","reviews": [{  "reviewer": "Reviewer1",  "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!"},{  "reviewer": "Reviewer2",  "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare."}]}
```

- Now, we want it to be able to hit the ratings service, so need the following env vars (todo: add link from docs or code):
  - `ENABLE_RATINGS`: true
- Checking the code, will also need to fix the hard coded 9080 port

```shell
oc patch dc reviews -p "$(cat .k8s/patches/3-dc-ratings.yml)"
buildconfig.build.openshift.io/reviews patched

oc patch bc reviews -p "$(cat .k8s/patches/4-bc-ref.yml)"
buildconfig.build.openshift.io/reviews patched

oc start-build reviews
build.build.openshift.io/reviews-3 started

curl "http://$host/reviews/1"
{"id": "1","reviews": [{  "reviewer": "Reviewer1",  "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!", "rating": {"stars": 5, "color": "black"}},{  "reviewer": "Reviewer2",  "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare.", "rating": {"stars": 4, "color": "black"}}]}
```

- Notice our reviews now include ratings
- Done!

## Product Page (Python)

```shell
cd ../productpage
git checkout openshift3
```

- try oc new-app

```shell
oc new-app 'python~https://github.com/stocky37/bookinfo-productpage.git#master' --name productpage
--> Found image 75e59ae (11 months old) in image stream "openshift/python" under tag "3.6" for "python"
...
--> Success

oc status
...
svc/productpage - x.x.x.x:8080
  dc/productpage deploys istag/productpage:latest <-
    bc/productpage source builds https://github.com/stocky37/bookinfo-productpage.git#master on openshift/python:3.6
    deployment #1 failed 18 minutes ago: config change
...

oc logs dc/productpage
ERROR: don't know how to run your application.
Please set either APP_MODULE, APP_FILE or APP_SCRIPT environment variables, or create a file 'app.py' to launch your application.
```

- needs a proper entrypoint
- [docs](https://github.com/sclorg/s2i-python-container/tree/master/3.6) tell us to use APP_SCRIPT (default app.sh) to run arbitrary script
- simple bash script will do, check out the code here: todo: link to bookinfo repo tag
- apply patch to new ref

```shell
oc patch bc productpage -p "$(cat .k8s/patches/1-bc-ref.yml)"
buildconfig.build.openshift.io/productpage patched

oc start-build productpage
build.build.openshift.io/productpage-2 started

oc status
...
svc/productpage - x.x.x.x:8080
  dc/productpage deploys istag/productpage:latest <-
    bc/productpage source builds https://github.com/stocky37/bookinfo-productpage.git#migrating-apps-v3/1.0 on openshift/python:3.6
    deployment #3 deployed about a minute ago - 1 pod
    deployment #2 deployed 18 minutes ago
    deployment #1 failed 39 minutes ago: config change
...
```

- that looks better, let's open the ui and see how if it works

![Product page error!](../../images/migrating-apps-to-openshift/productpage-error.png)

```shell
oc expose svc productpage
route.route.openshift.io/productpage exposed
```

- Look up the page, insert screenshot, notice errors
- Check the logs:

```shell
oc logs dc/productpage
...
DEBUG:urllib3.connectionpool:Starting new HTTP connection (1): details:9080
DEBUG:urllib3.connectionpool:Starting new HTTP connection (1): reviews:9080
DEBUG:urllib3.connectionpool:Starting new HTTP connection (1): reviews:9080
...
```

- Looks like we need to fix up the URLs
- Looks like the ports are currently harcoded to 9080, so we'll need to fix that
- May as well just hard code it for now, get it up and running and all that
- View the changes here: todo: add link to changes
- Apply the patch to the bc for latest version

```shell
oc patch bc productpage -p "$(cat .k8s/patches/2-bc-ref.yml)"
buildconfig.build.openshift.io/productpage patched

oc start-build productpage
build.build.openshift.io/productpage-3 started
```

- Test it out again in the browser

![Product page success!](../../images/migrating-apps-to-openshift/productpage-success.png)

- et voilà

## To Do

- Move envvars to deploymentconfig patch, ecept for reviews `MAVEN_S2I_ARTIFACT_DIRS`
- Double check repo links
- Make reviews use ratings service
- Link to actual istio documentation for envvars etc for the actual services

```shell
git co -b productpage master
git filter-branch -f --prune-empty --subdirectory-filter samples/bookinfo/src/productpage productpage
git remote add productpage git@stocky37.github.com:stocky37/bookinfo-productpage.git
git push productpage productpage:master
```
