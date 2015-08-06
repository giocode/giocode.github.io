---
layout: page
title: "projects"
date: 2015-07-15 13:36
comments: true
sharing: true
footer: true
---

Below is a list of the projects I have been working on recently.

### Anomaly detection (Mitacs research internship) 

Anomaly detection is an important analytics problem for business applications. It helps an organization in detecting operation bottlenecks and in reducing risks. However, when the number of metrics to monitor are large, it is crucial to develop efficient machine learning algorithms that are optimized for parallel processing. In this research project, I have been investigating parallel algorithms to enable fast and scalable predictive analytics. 

I developed new techniques for detecting data points in time series data that deviate from a normal behavior. In particular, I designed and implemented algorithms using time series decomposition, smoothing methods and robust statistics. The algorithms were prototyped in Matlab and implemented in Haskell. The benefits of these algorithms are that they are computationally efficient and can work on time series data that may exhibit seasonality and trend. In addition, they lead naturally to parallelization. Thus, they can be used for monitoring discrepancies in most time-series data. This project was generously supported by a D&B Cloud Innovation center and by a [Mitacs Accelerate](https://www.mitacs.ca/en/programs/accelerate) grant.


### Apache Spark projects

[Apache Spark](http://spark.apache.org/) is a cluster computing framework originally developed in the AMPLab at UC Berkeley. In contrast to Hadoop's MapReduce paradigm, Spark's in-memory processing provides performance up to 100 times faster for certain applications. By allowing user programs to load data into a cluster's memory and query it repeatedly, Spark is well-suited to various data applications such as machine learning, graph processing and real-time data analysis.

As the most active open-source project in Big Data, Spark has a large and diverse community. I have been involved with Spark since it was in incubation stage. The following list is a sample of my projects and contributions to this exciting project: 

- I created and co-organize the local [**Vancouver Spark Meetups**](http://www.meetup.com/Vancouver-Spark/), which now has over 300 members. 
- I also wrote a book on **Spark Graph Processing**. While waiting for its release, you can access a [**chapter preview here**](http://giocode.github.io/projects/spark-graph.html).
- **CPSC 546 - Optimization course project:** "_Distributed algorithms for training recommendation systems_" 
- Some Spark tutorials that I taught include: 
	- [IEEE Summer School on Signal Processing and Machine Learning for Big Data](https://sites.google.com/site/s3pbigdata2014/lecturers)
	- [Workshop on Distributed Machine Learning and Computing with Spark](http://www.meetup.com/Vancouver-Spark/events/178126142/) at SAP. 


### Deep learning for image classification (CPSC 540)

Deep learning and neural networks is a exciting subject not only for researchers but also for A.I. community in general. Google's acquisition of [DeepMind](http://deepmind.com/) is only one of the interesting developments in this field. Traditional approaches to machine learning have been based on having engineers build or select the right features for a learning problem. After extracting the features, the learning model needs to be trained with a large amount of labeled datasets.  Feature extraction is a hard topic especially for machine vision problem such as image classification. In contrast, neural networks can learn the features during the training - by themselves. However, care must be taken not overfit the model! In fact, neural networks are known to easily overfit since they can virtually approximate any function.

In my [machine learning course](http://www.cs.ubc.ca/~schmidtm/Courses/540-W16/) project, I investigated regularizations methods to prevent overfitting when training neural networks. Following this project, I wrote a [**compact and accessible tutorial**](http://giocode.github.io/projects/deep-learn.html) on this subject. It includes just the important math and contains illustrative code written in Julia language to explain the main concepts.


### Mobile app development

In my free time, I had the chance to learn and build mobile applications on both iOS and Android platform. In particular, I have published the [Hashter mobile app](https://play.google.com/store/apps/details?id=com.highhay.hashtag.free) for creating gorgeous social media posters.



