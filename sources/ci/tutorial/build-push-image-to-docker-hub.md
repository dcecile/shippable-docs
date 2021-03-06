page_description: Build and Push image to Docker Hub
main_section: CI
sub_section: Tutorials

# Build and Push Docker image to Docker Hub

This tutorial explains how to continuously build and push a Docker image to Docker Hub.

We will start with a simple Node.js application, run some CI tests and code coverage, and then package the application into a Docker image and push it to Docker Hub.

This document assumes you're familiar with the following concepts:

* [Docker build](https://docs.docker.com/engine/reference/commandline/build/)
* [Docker push](https://docs.docker.com/engine/reference/commandline/push/)
* [Docker login](https://docs.docker.com/engine/reference/commandline/login/)
* [Dockerfile](https://docs.docker.com/engine/reference/builder/)

If you're unfamiliar with Docker, you should start with learning how to implement this scenario manually. Refer to our blog for a step-by-step tutorial: [Build and push a Docker image to Docker Hub](http://blog.shippable.com/build-a-docker-image-and-push-it-to-docker-hub).

## Automated build and push to Docker Hub

There are many build services for Docker, but they offer limited functionality in addition to building an image. For example, you will not be able to run CI tests, code quality checks, code coverage, etc, before building your Docker image.

This tutorial will take you on a step-by-step journey on how to achieve this scenario.

<img src="/images/tutorial/build-push-docker-hub-fig1.png" alt="E2E Pipeline View">

Before you start, you should be familiar with the following Shippable platform features:

* [Shippable CI configuration](/platform/workflow/config/#ci-configuration)
* [Docker Registry integration](/platform/integration/dockerRegistryLogin)

### Instructions

The following sections explain the process of configuring a CI workflow to continuously test, build and push a Docker image to Docker Hub. We will use a Node.js app that uses Mocha and Istanbul for unit testing and code coverage.

**Source code is available at [devops-recipes/node_app](https://github.com/devops-recipes/node_app)**

**Complete YML is at [devops-recipes/node_app/shippable.yml](https://raw.githubusercontent.com/devops-recipes/node_app/master/shippable.yml)**

You can clone our sample repository to follow this tutorial.

####1. Add necessary Account Integrations

Integrations are used to connect Shippable Platform with external providers. More information about integrations is [here](/platform/tutorial/integration/howto-crud-integration/). We will use a Docker registry integration in this scenario.

#####1a. Add `Docker Registry` Integration**

To be able to push and pull images from Docker Hub, we add `drship_dockerhub ` integration. Make sure you name the integration `drship_dockerhub` since that is the name we're using in our sample automation scripts.

Detailed steps on how to add a Docker Registry Integration are [here](/platform/integration/dockerRegistryLogin/#creating-an-account-integration).

> Note: You might already have this if you have done any of our other tutorials. If so, skip this step

####2. Author CI configuration

The platform is built with "Everything as Code" philosophy, so all configuration is in a YAML-based file called **shippable.yml**.

Detailed CI configuration info is [here](/ci/yml-structure/).

#####2a. Add empty shippable.yml to your repo

Add an empty **shippable.yml** to the the root of your repository that contains your Dockerfile.

#####2b. Add CI config

Configure CI steps by adding the following code snipped to the YML file. If you're using a clone of our sample repository, the config is already present. You'll just need to make a few edits as noted below:

```
language: node_js

branches:
  only:
    - master

integrations:
  hub:
    - integrationName: drship_dockerhub     # integration name from step 1a
      type: dockerRegistryLogin

env:
  global:
    - TEST_RESULTS_DIR=$SHIPPABLE_REPO_DIR/shippable/testresults
    - CODE_COVERAGE_DIR=$SHIPPABLE_REPO_DIR/shippable/codecoverage
    - TESTS_LOC_DIR=$SHIPPABLE_REPO_DIR/tests
    - MOD_LOC=$SHIPPABLE_REPO_DIR/node_modules/.bin/
    - DOCKER_REPO="node_app"
    - DOCKER_ACC=devopsrecipes # replace with your Docker Hub account name    

build:
  ci:
    - shipctl retry "npm install"
    - mkdir -p $TEST_RESULTS_DIR && mkdir -p $CODE_COVERAGE_DIR
    - pushd $TESTS_LOC_DIR
    - $MOD_LOC/mocha --recursive "$TESTS_LOC_DIR/**/*.spec.js" -R mocha-junit-reporter --reporter-options mochaFile=$TEST_RESULTS_DIR/testresults.xml
    - $MOD_LOC/istanbul --include-all-sources cover -root "$SHIPPABLE_REPO_DIR/routes" $SHIPPABLE_REPO_DIR/node_modules/mocha/bin/_mocha -- -R spec-xunit-file --recursive "$TESTS_LOC_DIR/**/*.spec.js"
    - $MOD_LOC/istanbul report cobertura --dir $CODE_COVERAGE_DIR
    - popd
  post_ci:
    - docker build -t $DOCKER_ACC/$DOCKER_REPO:$BRANCH.$BUILD_NUMBER .
    - docker push $DOCKER_ACC/$DOCKER_REPO:$BRANCH.$BUILD_NUMBER

```

The above YML does the following things:

* Language is set to Node.js
* We will automatically build when a commit is made to `master` branch. Commits to other branches will not trigger a build.
* The `integration` created in first step is referenced here to automatically login to Docker Hub with your credentials from the integration
* Global environment variables store information about the Docker image. Please replace the `DOCKER_ACC` value with your account name on Docker Hub.
* The `build` section defines the scripts to build and push the image. As you can see, native docker commands can be directly used in your configuration.

####3. Enable the repo for CI

For automated builds, we need to enable the project for CI, which will automatically create the webhooks required to continuously build the project.

Detailed steps on enabling a repo are [here](/ci/enable-project/).

####4. Run your CI Project

You can either manually trigger your CI project or commit a change to the application git repository which should automatically build your project.

Verify that your image was pushed to Docker Hub!

## Adding CD capability

CI is a developer focused activity that produces a deployable unit of your application, a Docker image in this instance. If you want to add a Continuous Delivery workflow which deploys your application into successive environments like Test, Staging, Production and runs test suites, the downstream jobs performing these activities will need to know where to find your Docker image.

This section shows how you can output your Docker image information into an `image` resource which can be used by downstream jobs.

<img src="/images/tutorial/build-push-docker-hub-fig2.png" alt="E2E Pipeline View">


### Concepts

Here are the additional concepts you need to know before you start:

* [Shippable Assembly Line config](/platform/workflow/config/#assembly-lines-configuration)
* [Github integration](/platform/integration/github)
* [Resources](/platform/workflow/resource/overview/)
    * [image](/platform/workflow/resource/image)
    * [gitRepo](/platform/workflow/resource/gitrepo)
* [Jobs](/platform/workflow/job/overview/)
    * [runCI](/platform/workflow/job/runci)
* [Resources](/platform/workflow/resource/overview/)
    * [image](/platform/workflow/resource/image)


### Instructions

The following steps shows how to dynamically update the tags of image resource when the CI process that generates the image is finished.

**Source code is available at [devops-recipes/node_app](https://github.com/devops-recipes/node_app)**

**Complete YML is at [devops-recipes/node_app/shippable.yml](https://raw.githubusercontent.com/devops-recipes/node_app/master/shippable.yml)**

####1. Author Assembly Line configuration

Your Assembly Line config is also stored in **shippable.yml**, but the structure is quite different from CI config. Detailed Assembly Line configuration info is [here](/deploy/configuration).

#####1a. Add `resources` section of the config

`resources` section defines the image resource which holds information about the Docker image and its location.

```
resources:
  - name: node_app_img_dh
    type: image
    integration: drship_dockerhub # replace with your integration name
    versionTemplate:
      sourceName: "devopsrecipes/node_app" # replace with your Hub URL
      isPull: false
      versionName: latest
```

Here we are creating an image named `node_app_img_dh` that can be accessed using credentials from the integration `drship_dockerhub`. `versionTemplate` has the information used to create the first version of the image. `sourceName` contains the location of the image and the  `versionName` contains the tag. `isPull` is used to configure whether the image is automatically pulled or not. Each time a new tag is created, a new version of the resource will be created.

Detailed info about `image` resource is [here](/platform/workflow/resource/image).

#####1b. Add `jobs` section of the config

A job is an execution unit of the assembly line.

In order to treat your CI workflow like an Assembly Line job, i.e. be able to define `IN` and `OUT` resources, you need to define a `runCI` job with the name of your repository appended by `_runCI`.

In this section, we will define the `runCI` job that will specify `node_app_img_dh` as an `OUT`, meaning your CI workflow will update this resource.

Add the following at the very end of your **shippable.yml**:

#####i. job named `node_app_runCI`.

```
jobs:
  - name: node_app_runCI
    type: runCI
    steps:
      - OUT: node_app_img_dh
```

####2. Update node_app_img_dh in CI Config

To update the `node_app_img_dh` resource with artifact information, such as image name and tag, add the following to the `build` section of your **shippable.yml**:

```
env:
  global:
    #Add the resource name to env->global section     
    - SHIP_IMG_RES=$DOCKER_REPO"_img_dh"

build:
  # Add this on_success section to build section
  on_success:
    - shipctl put_resource_state $SHIP_IMG_RES versionName $BRANCH.$BUILD_NUMBER
```

* If the ci section runs without any error, then using in-built utility function `put_resource_state`, we copy the image name into the `sourceName` field, and image tag into the `versionName` field of image `node_app_img_dh` resource. Utility functions are invoked using the command `shipctl`. A full list of these commands are [here](platform/tutorial/using-shipctl)

####3. Push changes to shippable.yml

Commit and push all the above changes to **shippable.yml** .

####4. Add the Assembly Line to your Shippable Subscription

In order to tell Shippable to read your `jobs` and `resources` config, you need to add the repository containing the configuration as a "sync repository" by [following instructions here](/deploy/configuration/#adding-a-syncrepo). This automatically parses the `jobs` and `resources` sections of your **shippable.yml** config and adds your workflow to Shippable.

Your SPOG view should look something like this:

<img src="/images/tutorial/build-push-docker-image-fig4.png" alt="E2E Pipeline View">

####5. Test your Assembly Line

Add a commit to your repo and you should see the CI process kick off, which builds the Docker image file and then pushes it to Docker Hub. The image information is then pushed into the `node_app_img_dh` resource, which creates a new version of the resource.

If `node_app_img_dh` is an `IN` to another job in your workflow that pulls the Docker image, that job will be triggered every time the `node_app_img_dh` version is updated.

## Further Reading
* [Build and Push a Docker Image to Docker Hub](/ci/tutorial/build-push-image-to-docker-hub)
* [Working with Integrations](/platform/tutorial/integration/howto-crud-integration/)
* [Defining Resources in shippable.yml](/platform/tutorial/workflow/shippable-yml/#resources-config)
* [Defining Jobs in shippable.yml](/platform/tutorial/workflow/shippable-yml/#jobs-config)
* [Sharing information between Jobs](/platform/tutorial/workflow/share-info-across-jobs/)
