# Introduction

This project provides convenient Docker images for Katalon Studio and other Selenium-based testing frameworks, with following requirements:

* Images are easy to deploy and use for people with limited Docker knowledge,
* Up-to-date browser versions (Google Chrome, Mozilla Firefox) from official installation packages,
* Testing frameworks and companion tools are fully installed and configured for common use cases,
* Compatible with Cloud and local-based CIs.

At this moment, the following images are available:

* Base: contains common software for doing Selenium testing: Google Chrome, Mozilla Firefox, Xvfb, Java SE Runtime Environment (OpenJDK).
* [Katalon Studio](https://hub.docker.com/r/katalonstudio/katalon/): used for creating containers that can execute Katalon Studio tests and write reports to host's file system.

Versions of important packages is written in `/katalon/version` (or `$KATALON_VERSION_FILE`).

    cat $KATALON_VERSION_FILE
    Google Chrome 69.0.3497.100
    Mozilla Firefox 62.0
    Katalon Studio 5.7.1

# Building and Running Docker Images on Google Cloud
1. Clone this CURT Katalon Docker repository

	```bash
	git clone git@github.com:curt-labs/katalon-docker-images.git
	```

1. Copy the Katalon test suite code into a Katalon repository folder named `source` at the same level as the `src` folder
    > For example, `cp -R ~/repos/katalon-erp ~/repos/katalon-docker-images/katalon-circleci/source`
1. Make your changes as needed to the `Dockerfile`
1. Build the Docker image

	```bash
	docker build -t us.gcr.io/curt-yoda/yoda-katalon:single-suite .
	```

1. Test Locally

	```bash
	docker run us.gcr.io/curt-yoda/yoda-katalon:single-suite
	```

1. Tag the Image

	```bash
	docker tag us.gcr.io/curt-yoda/yoda-katalon:single-suite us.gcr.io/curt-yoda/yoda-katalon:latest
	```

1. Push the image to Google Cloud Container Registry

	```bash
	docker push us.gcr.io/curt-yoda/yoda-katalon:single-suite
	```

1. Redeploy the new image to the cluster and begin testing

	```bash
	kubectl set image deployment/katalon yoda-katalon=us.gcr.io/curt-yoda/yoda-katalon:latest
	```

> | <img src="https://gist.githubusercontent.com/lefte/a1f67432ad3588f5e46c28e900c842dd/raw/bca7c40cf7cfbb11aa95aefa2d8111cd376e2423/icons8-poison-windows10-100.png" height="24" valign="middle"> Warning |
|:--|
| Don't forget to kill the deployment when you want to stop testing, as it will keep restarting the test until told explicitly to stop |

## _One-Time Setup Tasks_
These are only needed if there isn't an existing cluster or deployment you want to use.

1. Authorize pushing to the Google Cloud Container Repository

	```bash
	gcloud auth configure-docker
	```

1. _(Once) Build a Kubernetes cluster_

	```bash
	gcloud beta container --project "curt-yoda" clusters create "yoda-katalon-test" --zone "us-central1-f" --no-enable-basic-auth --cluster-version "1.9.7-gke.6" --machine-type "g1-small" --image-type "COS" --disk-type "pd-standard" --disk-size "30" --scopes "https://www.googleapis.com/auth/compute","https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "1" --no-enable-cloud-logging --no-enable-cloud-monitoring --network "projects/curt-yoda/global/networks/default" --subnetwork "projects/curt-yoda/regions/us-central1/subnetworks/default" --addons HorizontalPodAutoscaling --enable-autoupgrade --enable-autorepair
	```

1. _(Once) Deploy an image_

	```yaml
	apiVersion: "extensions/v1beta1"
	kind: "Deployment"
	metadata:
	  name: "katalon"
	  namespace: "default"
	  labels:
	    app: "katalon"
	spec:
	  replicas: 3
	  selector:
	    matchLabels:
	      app: "katalon"
	  template:
	    metadata:
	      labels:
	        app: "katalon"
	    spec:
	      containers:
	      - name: "yoda-katalon"
	        image: "us.gcr.io/curt-yoda/yoda-katalon:single-suite"
	---
	apiVersion: "autoscaling/v1"
	kind: "HorizontalPodAutoscaler"
	metadata:
	  name: "katalon-hpa"
	  namespace: "default"
	  labels:
	    app: "katalon"
	spec:
	  scaleTargetRef:
	    kind: "Deployment"
	    name: "katalon"
	    apiVersion: "apps/v1beta1"
	  minReplicas: 1
	  maxReplicas: 5
	  targetCPUUtilizationPercentage: 80
	```

# Katalon Studio image

The container started from this image will expect following environment variables:
* `KATALON_OPTS`: all Katalon Studio console mode arguments except `-runMode`, `-reportFolder`, and `-projectPath`. For more details as well as an easy way to generate all arguments please refer to [the documentation](https://docs.katalon.com/display/KD/Console+Mode+Execution).

The following bind mounts should be used:

| Container's directory     | Host's directory  | Writable? |
| ------------------------- | ----------------- | --------- |
| /katalon/katalon/source | project directory | No - the source code will be copied to a temporary directory inside the container, therefore no write access is needed. |
| /katalon/katalon/report | report directory  | Yes - Katalon Studio will write execution report to this directory. |

If you need to configure proxy for Katalon Studio please use following parameters:

| Option Name          | Value Type | Values                              | Mandatory? |
| -------------------- | ---------- | ----------------------------------- | ---------- |
| proxy.option         | Fixed      | NO_PROXY, USE_SYSTEM, MANUAL_CONFIG | YES        |
| proxy.server.type    | Fixed      | HTTP, HTTPS, or SOCKS               | YES        |
| proxy.server.address | String     | Examples: locahost, 192.168.1.221   | YES        |
| proxy.server.port    | Integer    | Examples: 8888, 8080                | YES        |
| proxy.username       | String	    | Example: MyProxyUsername            | Optional (YES if your proxy server requires authentication) |
| proxy.password       | String     | Example: MyProxyPasswordOptional    | (YES if your proxy server requires authentication) |

These proxy information will be passed to browsers executing the tests.

For example, the following script will execute a project at `/home/ubuntu/katalon-test` and write reports to `/katalon/katalon/report`. Do not forget to put `--config` before the proxy configuration.

    #!/usr/bin/env bash

    katalon_opts='-browserType="Chrome" -retry=0 -statusDelay=15 -testSuitePath="Test Suites/TS_RegressionTest" --config -proxy.option=MANUAL_CONFIG -proxy.server.type=HTTP -proxy.server.address=192.168.1.221 -proxy.server.port=8888'
    docker run --rm -v /home/ubuntu/katalon-test:/katalon/katalon/source:ro -v /home/ubuntu/report:/katalon/katalon/report -e KATALON_OPTS="$katalon_opts" katalonstudio/katalon

Please visit https://github.com/katalon-studio/docker-images-samples for samples.

# Images built by community

We also host image built by community. If you want to add one, please fire a Pull Request. For example, `katalonstudio/katalon:contrib_PR_15` refers to the image built based on #15. We do not maintain or take responsiblity for any consequence made by using these images, so please use them at your own risk.
