---
layout: post
title:  "Thoughts on AWS Lambdas after 6 months of development"
date:   2019-10-06 10:55:26 -0700
categories: AWS Lambda
---

I've written three different projects in AWS Lambdas in Python, which have all been groupings of about around five to ten Lambdas. In each of these projects, I've exposed some of them as API Gateway REST endpoints and used the rest as "helper functions". I've used only a subset of the available AWS Lambda features, but I've employed a good chunk of them: SQS, DynamoDB, API Gateway, Redis (ElastiCache), VPC, Kinesis, and others. For all of my projects, I used Serverless with three plugins:

1. serverless-package-external
	* package common code (models, etc.)
2. serverless-python-requirements
	* plugin to package third party libraries
3. serverless-stage-manager
	* in order to deploy and manage different environments (local, prod, staging, etc.)


Overall, I've had a positive experience. The biggest advantages have been:
1. Scalability of Lambdas
	* probably what draws most people to Lambdas... essentially "limitless" concurrency
	* the bottleneck becomes other resources used (DBs, queues, etc.), most of which can be auto-scaled
	* concurrent execution can be explicitly controlled in each Lambda function
2. Naturally forces you into a "functional" style of programming
	* state has to be passed between Lambda functions or other resources like databases or queues
	* easy to unit test the functions individually
3. Ease of deployment
	* once you've configured the environment, you can rely on Serverless (or the SAM CLI) to deploy your changes
	* no need to configure and size EC2 or other similar resources
4. AWS ecosystem gives you "out of the box" tools to tackle many problems
	* queues, caches, streams, DBs of different types, etc.
	* events from many of these resources are easily configured to be a trigger for your Lambdas, which saves boilerplate coding and configuration

Downsides:
1. Newness and lack of examples
	* Since the framework is relatively new, there aren't as many examples out there as there are in typical web frameworks
2. Time spent in configuration and integration of Lambdas
	* Deploying a new project inevitably has required some churn to figure out permissions and integration of resources
3. Intermittent deployment errors with Serverless
	* I've run into some deployment issues that have been manifested as 403 errors, which have been challenging to debug at times
	* Most of these issues have been resolved by changing IAM permissions, which can be tricky at times to write

What I'd like to explore:
1. Local integration testing with Localstack
2. Using another language, like Go
3. Using the SAM CLI rather than Serverless to deploy
4. Deploying a Flask/Django app in AWS Lambdas using Zappa
5. I've only used Lambdas to write relatively small micro services; I'd be interested to hear what it's like to write larger services using AWS Lambdas.

Overall, I think AWS Lambdas delivers on its promise of allowing developers to write highly scalable applications with minimal configuration overhead and low computing cost.