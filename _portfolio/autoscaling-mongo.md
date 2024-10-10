---
title: "AutoScaling MongoDB Atlas"
excerpt: "Code for autoscaling MongoDB Atlas clusters based on a schedule. <br/><img src='/images/portfolio/golang-shirt-mongo.png' style='width:300px;'>"
collection: portfolio
tags:
  - mongodb
  - golang
---

The MongoDB Atlas Auto-Scaling project automates the scaling of MongoDB Atlas clusters based on a specific schedule, following custom business rules for different instance types, such as electable, read-only, and analytics nodes. Using AWS Lambda and EventBridge, this project allows for automatic adjustments to cluster specifications at predefined times, without relying on real-time resource usage.

## **Key Features**

**Schedule-based Scaling**: Automatically increase or decrease your MongoDB cluster's instance size according to scheduled times rather than resource utilization.

**AWS Lambda Integration**: The Lambda function is triggered by AWS EventBridge at scheduled times, enabling automated scaling during peak hours or other predefined periods.

**Synchronization Rules**: Certain parameters, such as disk type and IOPS, are synchronized across different instance types (electable, read-only, and analytics) to ensure consistency. These synchronization rules are in place to prevent the MongoDB Atlas API from returning errors when updating electable nodes, which could otherwise differ from the read-only nodes, potentially causing issues.

## **Main Use Cases**

**Peak Time Scaling**: Increase instance size during high-demand periods and revert to the original size outside of those times.

**Add Read-Only Nodes**: Add read-only nodes to optimize query capacity without impacting the primary nodes.

**Add Analytics Nodes**: Scale analytics nodes separately to handle large data volumes without affecting transactional operations.

**Adjust Disk IOPS**: Modify IOPS and disk type settings according to need, based on a predefined schedule.

## Payload Examples

*Peak Time Scaling*

```json
{
  "project": "project0",
  "cluster": "Cluster0",
  "electableSpecs": {
    "instanceSize": "M30",
    "AutoScale": {
      "MinInstanceSize": "M10",
      "MaxInstanceSize": "M30"
    }
  }
}
```

*Adding Read-Only Nodes*

```json
{
  "project": "project0",
  "cluster": "Cluster0",
  "readOnlySpecs": {
    "NodeCount": 2
  }
}
```

*Adding Analytics Nodes*

```json
{
  "project": "project0",
  "cluster": "Cluster0",
  "analyticsSpecs": {
    "NodeCount": 2,
    "instanceSize": "M30"
  }
}
```



[More information here](https://github.com/SamuelMolling/autoscaling-mongodb-atlas)
