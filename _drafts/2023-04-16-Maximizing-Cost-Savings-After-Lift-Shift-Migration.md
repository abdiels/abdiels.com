---
layout: post
title: "Maximizing Cost Savings After a Lift and Shift Migration"
subtitle: "to the the AWS cloud."
background: '/img/posts/aws-lift-and-shift/post-header.png'
---

## Introduction

In recent years, many organizations have migrated their applications and infrastructure to the cloud to capitalize on the numerous benefits, such as increased agility, scalability, and lower costs. A popular migration strategy is the "lift and shift" approach, where existing applications are moved to the cloud with minimal modifications. However, the work doesn't stop once the migration is complete. To truly maximize cost savings and efficiency, it's crucial to optimize your cloud resources post-migration.

In this blog post, we'll explore ways to save money after a lift and shift migration, focusing on systems with multiple EC2 instances and RDS nodes. By employing these strategies, you'll be able to achieve better resource utilization and lower costs in your cloud environment.

1. Right-sizing EC2 instances
One of the most effective ways to reduce costs is by right-sizing your EC2 instances. This involves selecting the appropriate instance type and size that matches your workload requirements. Monitor your instances' CPU, memory, and network usage to identify underutilized instances that can be downsized. Likewise, if you notice instances struggling to keep up with demand, you may need to upsize them for better performance.

2. Utilizing Reserved Instances and Savings Plans
AWS offers significant discounts for EC2 and RDS resources when you commit to using them for a longer period. By purchasing Reserved Instances (RIs) or subscribing to Savings Plans, you can achieve substantial savings compared to the on-demand pricing model. Evaluate your current and future workload requirements and choose a commitment plan that best suits your needs.

3. Optimizing RDS nodes
Similar to EC2 instances, right-sizing your RDS nodes is essential for cost optimization. Monitor database performance metrics, such as CPU utilization, connections, and storage, to determine if you can downsize your RDS nodes without compromising performance. Additionally, consider consolidating multiple databases on a single RDS instance to reduce costs.

4. Implementing Auto Scaling
Auto Scaling allows you to automatically adjust the number of EC2 instances or RDS nodes based on the demand. By implementing auto scaling, you can ensure that your infrastructure scales up during peak hours and scales down during low usage periods, reducing overall costs.

5. Leveraging Spot Instances
Spot Instances are spare EC2 instances that AWS offers at discounted rates. If your workloads can tolerate interruptions, consider using Spot Instances to save up to 90% compared to on-demand pricing. Be sure to implement a fallback strategy, such as using on-demand instances when Spot Instances are not available.

6. Deleting unused resources
As your cloud environment grows, it's crucial to regularly audit your resources and delete those that are no longer needed. This includes unused EC2 instances, EBS volumes, and RDS snapshots. Deleting unused resources helps to minimize storage costs and streamline your infrastructure.

7. Monitoring and reporting
Continuously monitor your cloud resources and their associated costs using tools like AWS Cost Explorer and Trusted Advisor. Set up billing alerts to keep track of your spending and identify cost-saving opportunities. Regular monitoring and reporting will help you maintain an optimized infrastructure and prevent cost overruns.

8. Using AWS Instance Scheduler
The AWS Instance Scheduler is a service that helps you reduce costs by allowing you to automatically shut down and start EC2 and RDS instances according to a predefined schedule. This is particularly useful for non-production environments, such as development and testing, where resources may not be needed outside of working hours.

To use the AWS Instance Scheduler, you'll need to:

a. Set up the Instance Scheduler infrastructure: Deploy the Instance Scheduler CloudFormation template in your AWS account, which creates the necessary resources, such as AWS Lambda functions, Amazon CloudWatch events, and Amazon DynamoDB tables.

b. Define schedules: Create schedules in DynamoDB that specify the desired start and stop times for your instances. You can create multiple schedules for different time zones, weekdays, or weekends.

c. Tag instances: Assign tags to your EC2 and RDS instances that correspond to the schedules you've created. The Instance Scheduler will use these tags to determine which instances should be started or stopped based on the active schedule.

d. Monitor and adjust: Regularly review the Instance Scheduler logs and metrics to ensure your instances are being started and stopped as intended. Adjust your schedules and tags as needed to optimize cost savings.

Keep in mind that there are some intricacies to consider when using the AWS Instance Scheduler:

- Ensure that your applications can gracefully handle the shutdown and restart of instances. For example, consider implementing health checks and retry mechanisms for database connections in your application code.
- Take note of any dependencies between instances, such as multi-tier applications, and ensure that they are started and stopped in the correct order.
- Be aware of potential data loss if you're using instance store volumes, as the data is not persistent across instance stops and starts.
- If you're using Spot Instances, be mindful that they may be terminated by AWS when there's a capacity shortage, even if the Instance Scheduler is set to keep them running.

By incorporating the AWS Instance Scheduler into your cost optimization strategy, you can further reduce costs by ensuring that instances are only running when they are needed. This, in combination with the other strategies mentioned in the blog post, will help you maximize cost savings and maintain an efficient cloud environment.

## Conclusion:

A lift and shift migration is just the beginning of your cloud journey. To fully leverage the cost-saving potential of the cloud, it's essential to optimize your resources post-migration. By right-sizing EC2 instances, utilizing reserved instances and savings plans, optimizing RDS nodes, and implementing auto scaling and spot instances, you can significantly reduce your cloud infrastructure costs. Regular monitoring and cleanup of unused resources will ensure that your cloud environment remains efficient and cost-effective in the long run.