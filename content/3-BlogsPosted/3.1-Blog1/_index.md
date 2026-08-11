---
title: "Blog 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# What Should an AI Engineer Prioritize When Learning AWS in the AWS Study Group?

After participating in the AWS Study Group, I found that the learning roadmap is quite comprehensive, covering foundational knowledge such as IAM, VPC, EC2, and S3, as well as AI/ML services such as Amazon SageMaker.

Initially, I thought I needed to learn everything in the prescribed order. However, since my goal is to become an AI Engineer, I asked myself a question:

_If my study time is limited, which topics should I prioritize to best support my future work in AI?_

After reviewing the entire roadmap and learning more about AWS services, here are the main conclusions I have drawn.

## First, AWS Fundamentals Are Still Very Important

I do not think that knowing Machine Learning alone is enough for an AI Engineer.

In practice, an AI model still needs data storage, a place to deploy the model, access control mechanisms, and various supporting services in order to operate effectively.

Therefore, services such as:

- Amazon S3
- IAM
- EC2
  are still topics that I plan to learn thoroughly.

However, if I had to prioritize them, I would spend more time studying AI/ML services.

## Amazon SageMaker – The Service I Prioritize the Most

This is probably the part of the AWS Study Group that I am most interested in.

Previously, when working on Machine Learning or Deep Learning projects, my workflow was usually:
**Dataset → Tiền xử lý dữ liệu → Train Model → Đánh giá → Lưu model**
Most of these steps were performed locally using PyTorch or Scikit-learn.

When learning about SageMaker, I realized that AWS integrates almost the entire workflow into a single platform.

During the Immersion Day workshop, I will have the opportunity to practice:

- Data preparation
- Feature Engineering
- Data analysis
- Training an XGBoost model
- Automatic Hyperparameter Tuning
- Deploying a model as an Endpoint

This helps me better understand how a Machine Learning model is developed and deployed in a real-world environment.

## Feature Engineering – More Than Just Data Processing

Previously, I mainly used Pandas or Scikit-learn for data preprocessing.

In SageMaker, AWS provides Data Wrangler to support data visualization, correlation analysis between features, and the development of Feature Engineering workflows before model training.

What I find particularly interesting is that processed data can be stored in Feature Store and reused by multiple models.

This is a concept that I have not had the opportunity to use in my personal projects, so I am very interested in learning more about it.

## Automatic Hyperparameter Tuning

When working with Deep Learning, I am sure many people have had to experiment with different values for:

- Learning Rate
- Batch Size
- Optimizer
- Epoch

Every time a value is changed, the model needs to be trained again.

With SageMaker, AWS provides Automatic Model Tuning, which automatically searches for a suitable set of Hyperparameters instead of requiring users to manually test each configuration.

In my opinion, this is a very useful feature when working with Machine Learning problems in real-world environments.

## Deploy Model – A Step That Many Students Often Overlook

I have noticed that when learning Machine Learning at university, most assignments usually end once the model achieves good Accuracy.

However, in a business environment, that is only one part of the process.

After training is completed, the model still needs to be deployed so that other applications can send data to it and receive prediction results.

Through SageMaker Endpoints, I have the opportunity to better understand how a model is put into practical use instead of simply running it on a personal computer.

In my opinion, this is an important skill for an AI Engineer.

## Topics I Will Still Learn

This does not mean that I will ignore the other modules.

Topics such as:

- Networking
- VPC
- IAM
- EC2
- Monitoring
  are all fundamental to building AI systems on AWS.

At this stage, however, I will prioritize learning the topics directly related to Machine Learning in greater depth. Afterwards, I will return to the remaining topics to develop a more comprehensive understanding.

## Finally

After reviewing the AWS Study Group roadmap, I realized that an AI Engineer today needs not only to know how to build models but also to understand how data is managed, how models are deployed, and how AI services operate in the Cloud.

That is also why I have chosen to prioritize topics related to Amazon SageMaker, Feature Engineering, Hyperparameter Tuning, and Model Deployment.

I hope that after completing these workshops, I will not only know how to train an AI model but also understand how to deploy that model to support real-world applications.

If you are also pursuing a career as an AI Engineer or Machine Learning Engineer, I would be happy to exchange ideas about how to learn AWS in a way that aligns with each person's career goals.

Thank you for reading!

## Link

<https://www.facebook.com/groups/awsstudygroupfcj/permalink/2224802604951366/?rdid=chYR8yCqrR14Gdm0#>
