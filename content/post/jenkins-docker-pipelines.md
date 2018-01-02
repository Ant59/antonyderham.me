---
title: "Declarative Jenkins Docker Pipeline for Angular CLI applications"
date: 2018-01-01
tags: [jenkins,docker,angular,pipeline]
---

## Purpose

For a while, I’ve used Drone CI to build my own projects at home. Drone provides a CI platform that uses Docker at it’s core to provide environments for build steps. This eliminates the need for any dependencies to be installed on the agents that are running the builds and provides a clean-room for each step of the build, and each build after it. For each stage in the pipeline, the platform pulls a Docker image, stands up an instance, binds the workspace, runs the commands in the pipeline definition, then tears down the container and begins the next step.

{{< figure src="/images/jenkins-docker-pipelines/pipeline.png" title="Pipeline Successful Output" >}}

At my company however, Jenkins is the CI platform of choice, and I wanted to be able to achieve the same benefits that Drone provides. Fortunately, Jenkins recently added Declarative Pipelines as a feature. This allows for pipelines as code, much like Drone. This means the steps to build a project are checked in as code alongside the project itself. This is a vast improvement over pre-defining the job through a UI. The pipelines-as-code strategy allows for version control, greater portability and the ability to change the build process on the fly as the project requires it. Ideal for verification of pull requests/changesets that need tweaks to the build.

The first project to undergo the move to this "pipeline-as-code" strategy is an Angular CLI application. This requires Node for build tools, Yarn for dependency management, and Java and Chromium for testing. My first attempt was to create a Dockerfile based on the official Node image (which already contains Node and Yarn), to install the other dependencies and use the resultant image as the build agent for each stage. Turns out that CircleCI provides Node images on Docker Hub with browsers pre-installed, giving me everything I needed without having to create my own image.

## Pipeline

Onto the pipeline itself. I wanted to provide everything that our current build setup provided and more. This meant pulling all dependencies for the project with Yarn, linting the TypeScript, running Jasmine unit tests with Karma, running E2E tests with Protractor, building the application with Angular CLI and wrapping the entire output as an Nginx-based Docker image to be pushed to our artefact repository. Each of these became a stage in the pipeline, most of which are simply invoking a package.json lifecycle script using Yarn.

{{< highlight groovy >}}
pipeline {
  agent none
  stages {
    // ... build stages here
  }
}
{{</ highlight >}}

Jenkins declarative pipelines begin with a root pipeline block. This block contains a stages block, in which all the build stages are defined. It also contains a top level agent declaration. For my pipeline here, each stage has it's own agent defined as later on in the process, I'll need to do some work directly with Docker outside of containers, to package the finished artefact and push to a registry. Therefore, the agent for the overall pipeline is defined as `none` to prevent Jenkins using an executors unecessarily.

### Fetch Dependencies

The first stage uses the [Yarn](https://yarnpkg.org) package manager tool to pull all the required Node package dependencies.

{{< highlight groovy >}}
stage('Fetch dependencies') {
  agent {
    docker 'circleci/node:9.3-stretch-browsers'
  }
  steps {
    sh 'yarn'
    stash includes: 'node_modules/', name: 'node_modules'
  }
}
{{</ highlight >}}

Firstly, the agent for this step is defined as a Docker image. As mentioned, I'm using the CircleCI images that include Node, Yarn and browsers. Jenkins will pull the image, launch a container on any slave that has available executors, bind the workspace and then execute the step commands within that container. If I wanted to restrict it to run on certain slaves only, I could define a label in the agent block too.

The steps in this stage are simple. Invoking `yarn` pulls all the packages. The stash step is used to allow later stages to run on other slaves and still have access to the packages from this stage. It took me more hours than I want to admit to discover stash as the solution to my workspace sharing problems. I originally had the pipeline restricted to run on a single slave only. Now that I can stash and unstash, I've been able to parallelise the testing steps for even faster build times.

### Lint

The next step is to check the TypeScript using the linter available with Angular CLI. This ensures at least a basic level of readability to the code being submitted.

{{< highlight groovy >}}
stage('Lint') {
  agent {
    docker 'circleci/node:9.3-stretch-browsers'
  }
  steps {
    unstash 'node_modules'
    sh 'yarn lint'
  }
}
{{</ highlight >}}

Again, the agent is defined with the same Docker image as before, and again this can run on any available executor on any slave.

The first step here is to unstash the `node_modules` that were saved in the first stage. This allows this stage to be run anywhere without the concern of not having dependencies downloaded to the workspace on any particular slave.

### Unit testing

The application I'm building uses the default Angular CLI test runner Karma with Jasmine unit tests.

{{< highlight groovy >}}
stage('Unit Test') {
  agent {
    docker 'circleci/node:9.3-stretch-browsers'
  }
  steps {
    unstash 'node_modules'
    sh 'yarn test:ci'
    junit 'reports/**/*.xml'
  }
}
{{</ highlight >}}

This stage is basically the same again. I have `test:ci` defined as a lifecycle script in my `package.json`, so it's as simple as invoking that command with Yarn.

{{<highlight json >}}
{
  "test:ci": "ng test --config karma.conf.ci.js --code-coverage --progress=false"
}
{{</ highlight >}}

The results are captured with the `junit` command to display test results within Jenkins.

### E2E testing

A non-standard approach is taken to E2E testing in this project. I am using Protractor with CucumberJS for behaviour driven development practices.

{{< highlight groovy >}}
stage('E2E Test') {
  agent {
    docker 'circleci/node:9.3-stretch-browsers'
  }
  steps {
    unstash 'node_modules'
    sh 'mkdir -p reports'
    sh 'yarn e2e:pre-ci'
    sh 'yarn e2e:ci'
    sh 'yarn e2e:post-ci'
    junit 'reports/**/*.xml'
  }
}
{{</ highlight >}}

The pre-ci step downloads the correct webdriver version for the version of Chromium installed in the Docker image. By default, CucumberJS outputs JSON results. The post-ci step converts this to JUnitXML for Jenkins to display test result information.

{{<highlight json >}}
{
  "e2e:pre-ci": "./node_modules/webdriver-manager/bin/webdriver-manager update --versions.chrome=2.34",
  "e2e:ci": "ng e2e --config protractor.conf.ci.js --proxy-config proxy.conf.ci.json --progress=false --webdriver-update false",
  "e2e:post-ci": "cat ./reports/e2e.json | ./node_modules/.bin/cucumber-junit > ./reports/e2e.xml"
}
{{</ highlight >}}

### Build

At last we're at the actual build stage. Once again, the same Docker image is used as the build agent and a Yarn lifecycle script is invoked. Angular CLI produces a production-targetted build of the application.

{{< highlight groovy >}}
stage('Compile') {
  agent {
    docker 'circleci/node:9.3-stretch-browsers'
  }
  steps {
    unstash 'node_modules'
    sh 'yarn build:prod'
    stash includes: 'dist/', name: 'dist'
  }
}
{{</ highlight >}}

The final step in this stage is to stash the output `dist/` directory containing the finished build output.

### Docker

The very last stage is to take the build output and package it as part of a Docker image to push to the registry as a finished artefact. The Dockerfile I've included as part of the project is baesd on the Nginx Alpine image. The resulting image can be deployed anywhere to serve the Angular app.

{{< highlight docker >}}
FROM nginx:1.13.1-alpine

EXPOSE 80

COPY dist /var/www
COPY config/nginx.conf /etc/nginx/nginx.conf
{{</ highlight >}}

A simple Nginx config is included which contains rules required for redirecting all URLs to `index.html`, where the built-in Angular Router can take over.

{{< highlight groovy >}}
stage('Build and Push Docker Image') {
  agent any
  environment {
    DOCKER_PUSH = credentials('docker_push')
  }
  steps {
    unstash 'dist'
    sh 'docker build -t $DOCKER_PUSH_URL/frontend .'
    sh 'docker login -u $DOCKER_PUSH_USR -p $DOCKER_PUSH_PSW $DOCKER_PUSH_URL'
    sh 'docker push $DOCKER_PUSH_URL/frontend'
  }
}
{{</ highlight >}}

This time, the steps take place on the bare slave. It is not possible to invoke Docker commands from within a Docker container on Jenkins without priviledge escalation. Therefore the agent here is defined as `any`.

First, the build output is unstashed. Then the Docker image is built from the Dockerfile. I'm making use of the `$DOCKER_PUSH_URL` variable which I have defined in the job itself. Next, the pipeline logs into the registry. In the `environment` block, I have used the `credentials` built-in function to define the variables of `$DOCKER_PUSH_USR` and `$DOCKER_PUSH_PSW`.

The completed image is then sent to the registry as a finished artefact.

## Putting it all together

Here is the entire pipeline from start to finish.

{{< highlight groovy "linenos=inline" >}}
pipeline {
  agent none
  stages {
    stage('Fetch dependencies') {
      agent {
        docker 'circleci/node:9.3-stretch-browsers'
      }
      steps {
        sh 'yarn'
        stash includes: 'node_modules/', name: 'node_modules'
      }
    }
    stage('Lint') {
      agent {
        docker 'circleci/node:9.3-stretch-browsers'
      }
      steps {
        unstash 'node_modules'
        sh 'yarn lint'
      }
    }
    stage('Unit Test') {
      agent {
        docker 'circleci/node:9.3-stretch-browsers'
      }
      steps {
        unstash 'node_modules'
        sh 'yarn test:ci'
        junit 'reports/**/*.xml'
      }
    }
    stage('E2E Test') {
      agent {
        docker 'circleci/node:9.3-stretch-browsers'
      }
      steps {
        unstash 'node_modules'
        sh 'mkdir -p reports'
        sh 'yarn e2e:pre-ci'
        sh 'yarn e2e:ci'
        sh 'yarn e2e:post-ci'
        junit 'reports/**/*.xml'
      }
    }
    stage('Compile') {
      agent {
        docker 'circleci/node:9.3-stretch-browsers'
      }
      steps {
        unstash 'node_modules'
        sh 'yarn build:prod'
        stash includes: 'dist/', name: 'dist'
      }
    }
    stage('Build and Push Docker Image') {
      agent any
      environment {
        DOCKER_PUSH = credentials('docker_push')
      }
      steps {
        unstash 'dist'
        sh 'docker build -t $DOCKER_PUSH_URL/frontend .'
        sh 'docker login -u $DOCKER_PUSH_USR -p $DOCKER_PUSH_PSW $DOCKER_PUSH_URL'
        sh 'docker push $DOCKER_PUSH_URL/frontend'
      }
    }
  }
}
{{< / highlight >}}
