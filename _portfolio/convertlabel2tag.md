---
title: "ConvertLabel2Tag"
excerpt: "Code for converting MongoDB Atlas labels to tags in dedicated clusters. <br/><img src='/images/golang-with-mongo.png'>"
collection: portfolio
tags:
  - mongodb
  - golang
---

ConvertLabel2Tag is code for converting MongoDB Atlas labels to tags in dedicated clusters. 

Previously, labels in MongoDB Atlas were only visible through the API, limiting their utility for management and visualization. With the introduction of tags, this information has become more accessible and visible directly in the MongoDB Atlas interface. 

This code creates tags based on the existing cluster labels. If a label does not exist, the code ignores the error and moves on to the next one.

In addition to being very simple to run.

I used golang and the MongoDB Atlas SDK to create this simple and functional solution that solved problems in some of the companies I worked at.

[More information here](https://github.com/SamuelMolling/convertLabel2Tag/tree/main)