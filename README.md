# Kubernetes Helm Tutorial

In this guide I will walk you through the basics of Kubernetes Helm. This
includes different kind of installation, the use of public charts and of course,
creating and managing your own charts.

Table of content:

+ [Introduction](#introduction)
+ [Part I: Setting up a local dev environment](part-01/README.md)
+ [Part II: Installing helm](part-02/README.md)
+ [Part III: Deploy your first chart](part-03/README.md)
+ [Part IV: Creating your own chart](part-04/README.md)
+ [Part V: Update, Rollback and what's under the hood](part-05/README.md)
+ [Part VI: Setup our own chart repo](part-06/README.md)
+ [Part VII: Dependency management](part-07/README.md)
+ [Part VIII: Multi-tenancy installation](part-08/README.md)
+ [References](#references)

## Introduction

Let's start off by shortly sum up what Helm is good for and why you should use
it. Helm makes your life easier when it comes to managing Kubernetes resources.
It helps you group your resources together to so called `Chart`s and assign them
a [semantic version](https://semver.org/). In that way you can package and
release you resources along with your application. To get more flexibility into
your resources, Helm has a very strong templating framework which is based on 
[Go-Template](https://golang.org/pkg/text/template/).

Enough talking let's make our hands dirty and start with setting up a local
dev environment.

## References

+ [Helm on Github](https://github.com/kubernetes/helm)
+ [Official Helm Site](https://helm.sh/)
+ [Chart best practices](https://github.com/kubernetes/helm/tree/master/docs/chart_best_practices)




