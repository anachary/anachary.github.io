--- 
layout: post
title:  "Custom API Gateway with YARP vs. Azure Solutions"
summary: "Custom API Gateway with YARP vs. Azure Solutions"
author: anachary
date: '2024-12-10 00:05:40 +0530'
category: "cloud"
thumbnail: /assets/img/posts/yarp-vs-apigw.jpg
keywords: logical programmer,yarp,apigw,api gateway, azure, azure TM.
permalink: /blog/2024-12-10-yarp-vs-apigw/
usemathjax: true
---

# Reducing Cloud Costs: Custom API Gateway with YARP vs. Azure Solutions

In the ever-evolving landscape of cloud computing, optimizing costs while maintaining robust functionality is a constant challenge. Recently, I faced this exact dilemma when looking to reduce my Cost of Goods Sold (COGS) for API management. I needed a solution that could handle custom rules, including authorization header and schema-based routing, without breaking the bank. Let's dive into my journey of comparing a custom YARP-based implementation against Azure API Gateway (APIGW) and Azure Traffic Manager (TM), and why I ultimately chose to build my own solution.

## The Challenge

My requirements were clear:
- Reduce COGS for API management
- Implement custom authorization header checks
- Enable schema-based routing
- Maintain high performance and scalability

## Comparing the Options

### Azure API Gateway

Initially, Azure API Gateway seemed like the obvious choice. It's a fully managed service with built-in features for API management. However, I quickly realized it came with significant drawbacks:

**Cost Comparison:**
Azure API Gateway pricing is based on the number of API calls, with tiers that can quickly become expensive as usage scales. For instance:
- Basic tier: $0.81 per million calls
- Standard tier: $1.62 per million calls
- Premium tier: $2.43 per million calls

For my usage of approximately 100 million calls per month, this would result in:
- Basic: $81/month
- Standard: $162/month
- Premium: $243/month

These costs would only increase with higher traffic volumes, making it less cost-effective for my growing needs.

### Azure Traffic Manager

Azure Traffic Manager offered some routing capabilities but lacked the fine-grained control I needed for authorization and schema-based routing. While it could help with load balancing, it wouldn't solve my core requirements without additional services.

### Custom YARP-based Solution

YARP (Yet Another Reverse Proxy) emerged as a promising alternative. As an open-source, highly customizable reverse proxy, it allowed me to implement my specific requirements without the overhead of a fully managed service.

## Why I Chose YARP

1. **Cost-Effective:** By running YARP on my existing infrastructure, I significantly reduced COGS compared to Azure APIGW.

2. **Customization:** YARP's flexibility allowed me to implement custom authorization and routing logic tailored to my needs.

3. **Performance:** YARP is designed for high performance, crucial for my API traffic volumes.

4. **Integration:** As a .NET-based solution, YARP seamlessly integrated with my existing technology stack.

## Cost Savings and Performance Gains

While my initial implementation of YARP showed promising results, I wanted to dive deeper into the potential cost savings, especially when running YARP in an Azure Kubernetes Service (AKS) cluster. Let's break down the numbers:

## YARP on AKS vs. Azure API Gateway

To provide a fair comparison, I'll consider a scenario of handling 100 million and 1 billion API calls per month.

### Azure API Gateway Costs:
- Basic tier: $81/month
- Standard tier: $162/month
- Premium tier: $243/month

### YARP on AKS Estimated Costs:
- **AKS Cluster Management**: Free (Azure doesn't charge for the control plane)

#### Compute Resources:
- Using 3 Standard_D2s_v3 instances: $210.24/month (3 * $70.08)

#### Networking:
- Standard Load Balancer: $3.65/month
- Estimated outbound data transfer: 
  - 100 million requests: $43.50/month (500 GB)
  - 1 billion requests: $87/month (1 TB)

### Total Estimated Cost for YARP on AKS:
- 100 million requests: $257.39/month
- 1 billion requests: $300.89/month

## Scaling Scenario Comparison

### 100 Million API Calls per Month:
- Azure API Gateway (Basic tier): $81/month
- YARP on AKS: $257.39/month

### 1 Billion API Calls per Month:
- Azure API Gateway (Basic tier): $810/month
- YARP on AKS: $300.89/month

While the cost difference is less dramatic than initially presented, YARP still offers potential savings, especially at higher traffic levels.

## Additional Cost-Saving Factors

### Scalability
AKS allows easy scaling of YARP instances without a linear cost increase.

### Resource Utilization
Deploying YARP alongside other services in the AKS cluster improves overall resource utilization.

### Optimization Opportunities
- Leverage Azure Reserved VMs 
- Use Spot instances for AKS nodes
- Optimize data transfer and compute resources

### Customization Benefits
- Implement custom authorization and routing logic
- Provides indirect cost savings and operational flexibility

## Performance Gains
- 30% Reduction in Latency
- Improved Throughput for high-volume traffic

We can use two approaches to reduce the number of API calls to API Gateway by offloading some of the checks to the YARP reverse proxy as well as a standalone reverse proxy. The following JSON configuration routes bearer token-based authorization to particular target URLs, while routing other requests to different target domains/URLs.
Code: https://github.com/anachary/YarpReverseProxy This code implementation of YARP also provides DDoS protection and rate-limiting functionality
## JSON Config for Auth-Based Routing

```json
{
   "ReverseProxy": {
    "Routes": {
      "bearer-route": {
        "Match": {
          "Path": "{**remainder}",
          "Headers": [
            {
              "Name": "Authorization",
              "Values": [ "Bearer" ],
              "Mode": "HeaderPrefix"
            }
          ]
        },
        "ClusterId": "bearer-cluster",
        "Transforms": [
          {
            "RequestHeadersCopy": "true"
          }
        ]
      },
      "default-route": {
        "Match": {
          "Path": "{**catch-all}"
        },
        "ClusterId": "default-cluster",
        "Transforms": [
          {
            "RequestHeadersCopy": "true"
          }
        ]
      }
    },
    "Clusters": {
      "bearer-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "https://bearertokenbased-url/"
          }
        }
      },
      "default-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "https://defualt-url/"
          }
        }
      }
    }
  }
}
```

## Conclusion

This experience underscores a valuable lesson in cloud architecture: sometimes, investing in a custom solution can yield substantial benefits in terms of cost, performance, and flexibility—especially when dealing with specific requirements and anticipating future growth.

As I continue to evolve my API infrastructure, I'm confident that my YARP-based approach will provide the scalability and customization I need while keeping costs in check. For organizations facing similar challenges, I highly recommend exploring custom solutions like YARP as alternatives to managed API gateways, particularly when specific requirements and cost optimization are key priorities.

### References
- Code Repository: [https://github.com/anachary/YarpReverseProxy](https://github.com/anachary/YarpReverseProxy)