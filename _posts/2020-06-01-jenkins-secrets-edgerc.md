---
layout: post
title: DevOps Secrets | Jenkins + Edgerc
image: /img/posts/2020/secrets/secret.png
tags: [akamai, devops, cicd, jenkins, automation]
---

TLDR; We have a lot of great tutorials on how to create a Jenkins Pipeline but I wanted to share how we can also store our `Edgerc` credentials file safely within Jenkins.

---
Instead of exposing our credentials on the pipeline script or adding it to the server we are going add the Edgerc file as a Secret file, this allows us to safely store and use them.

## Jenkins

Jenkins has the option for us to upload a file that will be stored and encrypted.
[You can read more about this here](https://www.jenkins.io/doc/book/using/using-credentials/)

This enables us to use our Edgerc file without exposing the credentials. For this example, we are using the Global Domain for them to be accessible to all Projects but this can be and should be limited to the scope of your pipeline.

> Jenkins >> Credentials >> Global Domain >> add Credentials

Here, all we need is to upload and provide an ID. This ID will be used with the pipeline.

![](/img/posts/2020/secrets/secret-file.jpg)

Once created you should have something like this.

![](/img/posts/2020/secrets/step-2.jpg)

## Pipeline Script

At this point, all we need is to use them. When modifying your script we will need to add Environment variables to help us the `Secret File`.

![](/img/posts/2020/secrets/pipeline-script.jpg)


Here is an example of how we will do this and if you pay attention you will see that we are calling the **Edgerc** via the credentials function and assigning its value to a variable called **Edgerc**.

```json

environment {
    edgerc = credentials('edgerc')
    template_config = 'gold.demo.gcs'
    pm_config = 'cli.demo'
  }

```

### How do we use it in our steps? 

Because CLI supports `--edgerc` as a way to indicate the path of the edge, we can use it to also pass the contents of the variable as seen below.

> akamai property --edgerc ${edgerc} retrieve ${template_config} --section default --file output/${template_config}.json

## Sample Script

This uses docker as the agent and this image has everything pre-installed.

* Fetch a configuration template
* Update a configuration with this template JSON.
* Push Configuration Change
* Activate Configuration version [Stage|Prod]

```

pipeline {
  agent {
    docker {
      image 'akamai/property:latest'
    }

  }
  stages {
    stage('Fetch Template Configuration') {
      steps {
        sh "akamai property --edgerc ${edgerc} retrieve ${template_config} --section default --file output/${template_config}.json"
        archiveArtifacts 'output/*.json'
      }
    }

    stage('Update Configuration') {
      steps {
        sh 'akamai property --edgerc ${edgerc} update ${pm_config} --file output/${template_config}.json --section default '
      }
    }

    stage('Activate [Staging]') {
      steps {
        sh 'akamai property --edgerc ${edgerc} activate ${pm_config} --network STAGING --email rmartine@akamai.com --section default '
      }
    }

    stage('Activate [Prod]') {
      steps {
        sh 'akamai property --edgerc ${edgerc} activate ${pm_config} --network PROD --email rmartine@akamai.com --section default '
      }
    }

  }
  environment {
    edgerc = credentials('edgerc')
    template_config = 'gold.demo.gcs'
    pm_config = 'cli.demo'
  }
}

```

**Have fun!...**

---