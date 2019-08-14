---
layout: post
title:  "Docker Continuous Security Scan"
date:  2019-08-14
desc: "Docker Continuous Security Scan"
keywords: "docker,security,dagda,anchore,clair,CI,harbor,quay"
categories: [Devops]
tags: [docker,security,dagda,anchore,clair,CI,harbor,quay]
icon: icon-html
---

As Docker containers become an almost common method of packaging and deploying applications, the instances of malware have increased. Securing Docker images is now a top priority. Docker image security scan is becoming a practice in most of the organizations (if not, it needs to be prioritized). It is recommended to include Docker images scan during execution of CI, like a continuous scan. To make it more elaborated, whenever new Docker images are build they should get scanned too.

In this post I will walk through couple of tools/methodologies that are well known for Docker image scan and their advantages/disadvantages.
This might be useful for making the decision of right choice of the tool.

------------------------------------------

## What Scan Does

------------------------------------------

Let's have a quick look into Docker image, what it is and how it built and run. To build a Docker image a `Dockerfile` is needed at first. `Dockerfile` contains all package/data/configuration needed for application to run, basically it is blue-print of application runtime environment. During container run, it loads all the package and start application as crafted in `Dockerfile`, the same way OS boots up and start the process.

For example I am building a NodeJS based UI application. To do that I need to have a `Dockerfile` as below -

```dockerfile
# Base Image
FROM nginx:alpine

# Additional Packages
RUN apk --no-cache add openssl

# Additional data
COPY nginx.conf /etc/nginx/nginx.conf

# Application Code
WORKDIR /usr/share/nginx/html
COPY dist/sample-hello-world/ .
```

Once I build the Docker image using the above mentioned `Dockerfile`, it basically creates an image the below types of layers -

```yaml
layer 0: FROM alpine:3.10
layer 1: ENV NGINX_VERSION=1.15.12
layer 2: RUN set -x && addgroup -g 101 -S nginx ...
layer 3: RUN apk --no-cache add openssl
layer 4: COPY file:4ed084f4e0006dfb1eacf439a519b4e6323a7878e38abff318b69d02dec61999 in /etc/nginx/nginx.conf
layer 5: WORKDIR /usr/share/nginx/html
layer 6: COPY dir:1d2d33ae85d9f689da1ff50a52376e8b6d6a9b8d9fbe74c39a40c7038eb598c2 in .
layer 7: EXPOSE 80
layer 8: CMD ["nginx" "-g" "daemon off;"]
```

After build once the image scan initiated, it will find the base image hierarchy and get all the different packages installed/copied/downloaded along with version in the image and search for publicly known vulnerabilities in CVE databases for that specific OS family.

However it is important to note what all the scan can not cover, it might not cover anything related to open ports or networking security, they are scanned by any runtime scanner like [Cilium](https://cilium.io/). Also it will not be able to highlight anything related to privileged permission, shared resources, volumes, etc.

Even I have seen common issues with not identifying vulnerabilities in downloaded packages, they are majorly because of renaming the downloaded packages and removing the version information's.

------------------------------------------

## In-build Scan in Registry

------------------------------------------

There are some Docker on-premises/enterprise registry providers like [Harbor](https://goharbor.io/) - a CNCF project, [QUAY](https://coreos.com/quay-enterprise/) - a CoreOS Product, [Docker Enterprise Hub](https://www.docker.com/products/docker-hub), etc. who felt Docker image scan is a key aspect of a registry and made scan as a built-in feature.

Among them Harbor and QUAY use Clair, and I am not sure yet about Docker Hub Security scan tool.

Scanning in Registry is very simple, most of the cases nothing needs to be done additionally other than publishing the image and based on the process mentioned in respective documentation the scan will be triggered and also reports can be viewed.

In some registry solution like Harbor supports additional features to invoke on-demand scan, reports via API (needed to enforce policy, I will discuss that later in this post), etc. Check the document of Harbor for more details.

------------------------------------------

## Scan in Dedicated tools

------------------------------------------

There are several open-source/commercial tools are available to perform Docker image security scan. I will try to cover about the tools I have came across so far. Please note I am keeping open-source community in mind :). Hence some of the disadvantages can be ruled-out if you approach to enterprise offering of the tools.

### [Clair](https://coreos.com/clair/docs/latest/)

**Advantages:**

* Most used open-source tools for scanning, inbuilt availability in Docker registry tools like Harbor/Quay
* Strong community support
* CI/CD Integration
* Easy setup
* Completely open-source and can be extended easily

**Disadvantages:**

* Week vulnerability database
* Package detection limited to only Package managers (dpkg/rpm/apk/etc.)
* Not much things can be done by API's
* Lack of documentation
* No Kubernetes integration

### [Anchore](https://anchore.com/)

**Advantages:**

* Widely used by several companies like Cisco/Microsoft/eBay/Atlassian, etc.
* Strong CVE database
* Support for user defined policy enforcement, very important for enterprises
* Kubernetes integration using an admission controller, the much needed in recent days
* CI/CD Integration
* Good API support
* Easy to install & maintain
* Supports software packages along with os packages

**Disadvantages:**

* No graphical UI in open-source version
* Several features like Policy GUI editor, RBAC, air-gapped data feed, etc. are available in Enterprise only
* No graphical reports in open-source version
* There are few false positives in vulnerabilities finding (based on some sample images I have used to scan)

### [Aqua](https://www.aquasec.com/)

**Advantages:**

* Widely used by several companies
* Very strong CVE database (honestly I was impressed on the findings)
* CI/CD Integration
* Very easy to use as it does not need any installation (open-source)
* Not many false positives in vulnerabilities finding (based on some sample images I have used to scan)

**Disadvantages:**

* No graphical UI in open-source version
* Almost all the features like Malware scan, package scan K8s integration, etc. are available in Enterprise only
* In open-source version there is no clear indication on scan limits - I was getting `Your token will be rate-limited to a reasonable number of scans`
* No centralized reporting database in open-source version, hence integration with Kubernetes is not possible

### [Dagda](https://github.com/eliasgranderubio/dagda)

**Advantages:**

* New tool but it is coming in the comparison study with other tools.
* Very strong CVE database
* CI/CD Integration
* Easy/Cheaper to maintain and install as it does not involve much storage
* Support many software packages (Java/NodeJS/Python/etc.) along with os packages

**Disadvantages:**

* No graphical UI
* No support for user defined policy enforcement
* No Kubernetes integration
* There are many false positives in vulnerabilities finding (based on some sample images I have used to scan)

------------------------------------------

## Factors to be considered for choosing the right tool

------------------------------------------

While choosing the appropriate tool from the list above, you may keep in mind the following points - (I have tried the best fit according to me for each points)

* **Scale of Business:** For a small scale organization (~10 docker images) some registry solutions like Docker Enterprise Hub (cloud) or Harbor (OnPrem) might work well as not much extra attention needed for image security scan.
* **Open Source Community driven:** If you want to choose purely community driven open source, you may go for Anchore or Clair or Dagda. Make sure to articulate the cost/effort needed to address the missing features that you need.
* **Custom Policy:** Anchore is a clear winner here, you can have several meaningful policies to be enforced, like password in container, some vulnerable files in container (`~/.aws/credentials`) etc.
* **Security Gateway in runtime:** Anchore again is a clear winner here (other tools like Dagda/Clair can be extended with some effort). With Kubernetes admission controller support it works well as a security gateway in Kubernetes. For example if some images are found to be vulnerable or violate custom policy, the respective Pod can be restricted to run in Kubernetes cluster (several customization can be done w.r.t. the gateway configuration)
* **Effort to address:** This is one of the important factor we often ignore while choosing any tool. After getting to know about vulnerabilities how costly (effort wise) it is to address them, here false positives plays a role, hence Aqua might be a possible tool as it is having less false positives

------------------------------------------

## What after scan

------------------------------------------

Wow you are reading this means, now you have some tool in your mind and ready for next steps. Tools can keep reporting vulnerabilities, that help certainly but unless someone fix them appropriately it will not add much values w.r.t. security of the application.

To address any reported vulnerabilities, you need to identify the type, like OS or Software. If it is OS and coming from base image, then some later/latest version of base image must be tried. If does not solve, the specific vulnerable package can be upgraded alone to version where it is fixed. Same goes for Software packages also.

Let's have a look at below example of how we can address the fix of a OS package coming from base image.

```dockerfile
FROM nginx:alpine

# Fixing CVE-2019-11068 vulnerability reported by Anchore
RUN apk --no-cache upgrade libxslt

COPY nginx.conf /etc/nginx/nginx.conf

WORKDIR /usr/share/nginx/html
COPY dist/cicd-anchore-portal/ .
```

---