---
title: Jenkins pipeline as a service
date: 2022-08-16 11:00:00 +0100
categories: [DevOps]
tags: [cicd, jenkins, devops]
description: How to write reusable Jenkins piplelines and automatically use them in hundreds of projects using Jenkins Shared Libraries and plugins.
---
If you are working in software development, there is a big chance that you are using Jenkins for Continuous Integration. It's a great, powerful tool. Unfortunately, it also has many issues:
* **Managing pipelines across multiple projects can be a mess.** Usually, there's a lot of copy-paste code in Jenkinsfiles. You have to edit them separately to add a new feature or fix a bug.
* **It's hard to enforce using best practices.** There are dozens of ways to write the same simple task.
* **Pipelines code can be insecure.** For example, pipelines written by inexperienced developers can have credentials leaks.
* **Developers don't like writing CI code.** They want to focus on application code, especially when they don't work with JVM languages (Jenkins pipelines use groovy-based DSL).

In this article, I would like to explain how to manage CI code in one place and reduce Jenkinsfile size to a few lines.

## Shared Libraries - all pipelines in one place

[Jenkins Shared Libraries](https://www.jenkins.io/doc/book/pipeline/shared-libraries/) is the tool we need to solve the issues mentioned. We can use them to put reusable pipeline code in one repository and share it across many projects.

The structure of the library is quite simple.

```
+- src                          # Groovy source files
|   +- package
|       +- UtilClass.groovy     # E.g. we can put common logic for artifacts versioning here
+- vars                         # Global variables
|   +- someStep.groovy          # We can define reusable step
|   +- thePipeline.groovy          # Or we can define whole pipeline here
+- resources                    # static resource files to use in library
|   +- Dockerfile.tpl           # E.g. we can put template for Dockerfiles here
|   +- sonar-project.properties # Or put SonarQube configuration and inject it in every project
```

There's excellent documentation on the official website about [defining](https://www.jenkins.io/doc/book/pipeline/shared-libraries/#defining-shared-libraries) and [configuring](https://www.jenkins.io/doc/book/pipeline/shared-libraries/#using-libraries) libraries in Jenkins. Therefore I will focus on practical use. Our goal is to reduce Jenkinsfile in every project to a few lines.

```groovy
@Library('ourlib') _
thePipeline()
```

To achieve this, we put the whole pipeline code in a global variable defined in `var/thePipeline.groovy`.

```groovy
def call(Map config) {
  pipeline {
    stages {
      stage('Checkout') {
        steps {
          checkout scm
        }
      }
      stage('Build') {
        steps {
          // Building artifacts
          // Automatic versioning e.g. based on git tags and branch names
        }
      }
      stage('Test') {
        steps {
          // Running tests
          // Static code analysis
          // Publishing test reports
        }
      }
      stage('Publish') {
        steps {
          // Publishing artifacts in artifact repository
        }
      }
    }
  }
}
```

This simple technique allows us to maintain all CI code in a project or even an organization in one place. We can easily add new features to hundreds of pipelines by releasing a new library version. I prefer configuring Jenkins to use the master branch for every shared library. Every new feature merged to master in a library repo is automatically available in pipelines.

### Customization

But what if we have to pass additional information to the pipeline? Or maybe we need to perform some custom actions before build? Because we are using Groovy, it's simple. We can pass params with additional configuration or even scripts as Closure objects.

```groovy
// Jenkinsfile
@Library('ourlib') _
thePipeline(
  docker: true,
  beforeBuild: { args ->
    // Our custom code
  }
)
```

And read everything in library as Map.
```groovy
// thePipeline.groovy
def call(Map config) {
  pipeline {
    stages {
      ...
      stage('Build') {
        steps {
          if(config.beforeBuild) {
            final someArgsToPassToScript = ...
            config.beforeBuild(someArgsToPassToScript)
          }

          // Building artifacts

          if(config.docker) {
            // Building Docker image
          }
        }
      }
      ...
    }
  }
}
```

And what if we have projects with different stacks and all pipeline steps differ? I prefer defining one generic base pipeline with skeleton and core functionalities. Then, I define child pipelines with customizations. It can look like below.

```
+- vars
|   +- thePipeline.groovy    # Base pipeline
|   +- mavenPipeline.groovy  # Pipeline for Maven projects
|   +- npmPipeline.groovy    # Pipeline for NPM projects
|   +- mlPipeline.groovy     # Pipeline for dockerized Machine Learning models trained on k8s
```

```groovy
// thePipeline.groovy
def call(Map config) {
  pipeline {
    stages {
      stage('Checkout') {
        steps {
          checkout scm
        }
      }
      stage('Build') {
        when {
          config.buildScript
        }
        steps {
          // Some common code
          config.buildScript(argsToPass)
          // Some common code
        }
      }
      stage('Test') {
         when {
          config.testScript
        }
        steps {
          // Some common code
          config.testScript(argsToPass)
          // Some common code
        }
      }
      ...
    }
  }
}

//mavenPipeline.groovy
def call(Map config) {
  thePipeline(
    buildScript: { args ->
      sh 'mvn clean package'
      // Other Maven specific code
    },
    testScript: { args ->
      // Running tests with Maven
    }
  )
}

//npmPipeline.groovy
def call(Map config) {
  thePipeline(
    buildScript: { args ->
      // Building artifacts with npm
    },
    testScript: { args ->
      // Running tests with npm
    }
  )
}
```

This technique helps us to maximize code reusability and keep our CI code clean.

## Summary

And that's it! By moving pipeline code to shared library, we have achieved many benefits.
* We maximized code reusability.
* We delivered ready to use CI processes to application developers.
* We centralized maintenance of pipelines.

### Consequences of moving pipelines to a library

We must also bear in mind some consequences.
* **We need an owner of the library.** There should be one team responsible for maintenance. Changes can be developed by external teams, but someone has to conduct code review and accept pull requests.
* **Changes in library affect many projects.** We should test everything very carefully before release. Having automated regression tests is highly recommended.
* **All changes should be backward compatible.**
* **We should maintain a transparent changelog** and keep our users informed about changes.

In my opinion, the pros far outweigh the cons. I encourage you to experiment with shared libraries. You will fall in love with them very quickly. See [the documentation](https://www.jenkins.io/doc/book/pipeline/shared-libraries/#writing-libraries) for more advanced techniques.


