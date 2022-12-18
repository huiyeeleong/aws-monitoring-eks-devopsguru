Solution Overview
In this post, we will demonstrate the new Amazon DevOps Guru features around cluster grouping and additionally supported Amazon EKS metrics. To demonstrate these features, we will show you how to create a Kubernetes cluster, instrument the cluster using AWS Distro for OpenTelemetry, and then configure Amazon DevOps Guru to automate anomaly detection of EKS metrics. A previous blog provides detail on the AWS Distro for OpenTelemetry collector that is employed here.

Prerequisites
Install eksctl for creating Amazon Elastic Kubernetes Service Cluster
Install kubectl for managing Amazon Elastic Kubernetes Cluster
Amazon Elastic Kubernetes Service(EKS)
AWS Distro for OpenTelemetry
Amazon DevOps Guru
Amazon Simple Notification Service(SNS)
EKS Cluster Creation
We employ the eksctl CLI tool to create an Amazon EKS. Using eksctl, you can provide details on the command line or specify a manifest file. The following manifest is used to create a single managed node using Amazon Elastic Compute Cloud (EC2), and this will be created and constrained to the specified Region via entry metadata/region and Availability Zones via the managedNodeGroups/availabilityZones entry. By default, this will create a new VPC with eight subnets.

# An example of ClusterConfig object using Managed Nodes
---
    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig

    metadata:
      name: devopsguru-eks-cluster
      region: <SPECIFY_REGION_HERE>
      version: "1.21"

    availabilityZones: ["<FIRST_AZ>","<SECOND_AZ>"]
    managedNodeGroups:
      - name: managed-ng-private
        privateNetworking: true
        instanceType: t3.medium
        minSize: 1
        desiredCapacity: 1
        maxSize: 6
        availabilityZones: ["<SPECIFY_AVAILABILITY_ZONE(S)_HERE"]
        volumeSize: 20
        labels: {role: worker}
        tags:
          nodegroup-role: worker
    cloudWatch:
      clusterLogging:
        enableTypes:
          - "api"
To create an Amazon EKS cluster using eksctl and a manifest file, we use eksctl create as shown below. Note that this step will take 10 – 15 minutes to establish the cluster.
$ eksctl create cluster -f devopsguru-managed-node.yaml
2021-10-13 10:44:53 [i] eksctl version 0.69.0
…
2021-10-13 11:04:42 [✔] all EKS cluster resources for "devopsguru-eks-cluster" have been created
2021-10-13 11:04:44 [i] nodegroup "managed-ng-private" has 1 node(s)
2021-10-13 11:04:44 [i] node "<ip>.<region>.compute.internal" is ready
2021-10-13 11:04:44 [i] waiting for at least 1 node(s) to become ready in "managed-ng-private"
2021-10-13 11:04:44 [i] nodegroup "managed-ng-private" has 1 node(s)
2021-10-13 11:04:44 [i] node "<ip>.<region>.compute.internal" is ready
2021-10-13 11:04:47 [i] kubectl command should work with "/Users/<user>/.kube/config"
Once this is complete, you can use kubectl, the Kubernetes CLI, to access the managed nodes that are running.
$ kubectl get nodes
NAME STATUS ROLES AGE VERSION
<ip>.<region>.compute.internal Ready <none> 76m v1.21.4-eks-033ce7e
AWS Distro for OpenTelemetry Collector Installation
We will use AWS Distro for OpenTelemetry Collector to extract metrics from a pod running in Amazon EKS. This will collect metrics within the Kubernetes cluster and surface them to Amazon CloudWatch. We start by defining a policy to allow access. The following information comes from the post here.

Attach the CloudWatchAgentServerPolicy IAM Policy to worker node
Open the Amazon EC2 console.
Select one of the worker node instances, and choose the IAM role in the description.
On the IAM role page, choose Attach policies.
In the list of policies, select the check box next to CloudWatchAgentServerPolicy. You can use the search box to find this policy.
Choose Attach policies.
Deploy AWS OpenTelemetry Collector on Amazon EKS
Next, you will deploy the AWS Distro for OpenTelemetry using a GitHub hosted manifest.

Deploy the artifact to the Amazon EKS cluster using the following command:
$ curl https://raw.githubusercontent.com/aws-observability/aws-otel-collector/main/deployment-template/eks/otel-container-insights-infra.yaml | kubectl apply -f -
View the resources in the aws-otel-eks namespace.
$ kubectl get pods -l name=aws-otel-eks-ci -n aws-otel-eks
NAME READY STATUS RESTARTS AGE
aws-otel-eks-ci-jdf2w 1/1 Running 0 107m
View Container Insight Metrics in Amazon CloudWatch
Access Amazon CloudWatch and select Metrics, All metrics to view the published metrics. Under Custom Namespaces, ContainerInsights is selectable. Under this, one can view metrics at the cluster, node, pod, namespace, and service granularity. The following example shows pod level metrics of CPU:

The AWS Console with Amazon Cloudwatch Container Insights Pod Level CPU Utilization.

Amazon Simple Notification Service
It is necessary to allow Amazon DevOps Guru access to Amazon SNS in order for Amazon SNS to publish events. During the setup process, an Amazon SNS Topic is created, and the following resource policy is applied:

{
    "Sid": "DevOpsGuru-added-SNS-topic-permissions",
    "Effect": "Allow",
    "Principal": {
        "Service": "region-id.devops-guru.amazonaws.com"
    },
    "Action": "sns:Publish",
    "Resource": "arn:aws:sns:region-id:topic-owner-account-id:my-topic-name",
    "Condition" : {
      "StringEquals" : {
        "AWS:SourceArn": "arn:aws:devops-guru:region-id:topic-owner-account-id:channel/devops-guru-channel-id",
        "AWS:SourceAccount": "topic-owner-account-id"
    }
  }
}
Amazon DevOps Guru
Amazon DevOps Guru can now be leveraged to monitor the Amazon EKS cluster and Managed Node Group. Select Amazon DevOps Guru, and select Get started as shown in the following figure to do this.

The Amazon DevOps Guru service via the AWS Console.

Once selected, the Get started console displays, letting you specify the IAM role for DevOps guru to access the appropriate resources.

The Get started dialog for Amazon DevOps Guru including instructions on how the service operates, IAM Role Permissions and Amazon DevOps Guru analysis coverage.

Under the Amazon DevOps Guru analysis coverage, Choose later is selected. This will let us specify the CloudFormation stacks to monitor. Select Create a new SNS topic, and provide a name. This will be used to collect notifications and allow for subscribers to then be notified. Select Enable when complete.

The Amazon DevOps Guru analysis coverage allowing the user to select all resources in a region or to choose later. In addition the image shows the dialog that requests the user specify an Amazon SNS topic for notification when insights occur.

On the Manage DevOps Guru analysis coverage, select Analyze all AWS resources in the specified CloudFormation stacks in this Region. Then, select the cluster and managed node group AWS CloudFormation stacks so that DevOps Guru can monitor Amazon EKS.

A dialog where the user is able to specify the AWS CloudFormation stacks in a region for analysis coverage. Two stacks are select including the eks cluster and eks cluster managed node group.

Once this is selected, the display will update indicating that two CloudFormation stacks were added.

Amazon DevOps Guru Settings including DevOps Guru analysis coverage and Amazon SNS notifications.

Amazon DevOps Guru will finally start analysis for those two stacks. This will take several hours to collect data and to identify normal operating conditions. Once this process is complete, the Dashboard will display that those resources have been analyzed, as shown in the following figure.

The completed analysis by DevOps guru of the two AWS Cloudformation stacks indicating a healthy status for both.

Enable Encryption on Amazon SNS Topic
The Amazon SNS Topic created by Amazon DevOps Guru will not enable encryption by default. It is important to enable this feature to encrypt notifications at rest. Go to Amazon SNS, select the topic that is created and then Edit topic. Open the Encryption dialog box and enable encryption as shown in the following figure, specifying an alias, or accepting the default.

The Encryption dialog for Amazon SNS topic when it is Edited.

Deploy Sample Application on Amazon EKS To Trigger Insights
You will employ a sample application that is part of the AWS Distro for OpenTelemetry Collector to simulate failure. Using the following manifest, you will deploy a sample application that has pod resource limits for memory and CPU shares. These limits are artificially low and insufficient for the pod to run. The pod will exceed memory and will be identified for eviction by Amazon EKS. When it is evicted, it will attempt to be redeployed per the manifest requirement for a replica of one. In turn, this will repeat the process and generate memory and pod restart errors in Amazon CloudWatch. For this example, the deployment was left for over an hour, thereby causing the pod failure to repeat numerous times. The following is the manifest that you will create on the filesystem.

kind: Deployment
apiVersion: apps/v1
metadata:
  name: java-sample-app
  namespace: aws-otel-eks
  labels:
    name: java-sample-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: java-sample-app
  template:
    metadata:
      labels:
        name: java-sample-app
    spec:
      containers:
        - name: aws-otel-emitter
          image: aottestbed/aws-otel-collector-java-sample-app:0.9.0
          resources:
            limits:
              memory: "128Mi"
              cpu: "200m"
          ports:
          - containerPort: 4567
          env:
          - name: OTEL_OTLP_ENDPOINT
            value: "localhost:4317"
          - name: OTEL_RESOURCE_ATTRIBUTES
            value: "service.namespace=AWSObservability,service.name=CloudWatchEKSService"
          - name: S3_REGION
            value: "us-east-1"
          imagePullPolicy: Always
To deploy the application, use the following command:

$ kubectl apply -f <manifest file name>
deployment.apps/java-sample-app created
Scenario: Improved context from DevOps Guru Container Cluster Grouping and Increased Metrics
For our scenario, Amazon DevOps Guru is monitoring additional Amazon CloudWatch Container Insight Metrics for EKS. The following figure shows the flow of information and eventual notification of the operator, so that they can examine the Amazon DevOps Guru Insight. Starting at step 1, the container agent (AWS Distro for OpenTelemetry) forwards container metrics to Amazon CloudWatch. In step 2, Amazon DevOps Guru is continually consuming those metrics and performing anomaly detection. If an anomaly is detected, then this generates an Insight, thereby triggering Amazon SNS notification as shown in step 3. In step 4, the operators access Amazon DevOps Guru console to examine the insight. Then, the operators can leverage the new user interface capability displaying which cluster, namespace, and pod/service is impacted along with correlated Amazon EKS metric(s).

 The flow of information and eventual notification of the operator, so that they can examine the Amazon DevOps Guru Insight. Starting at step 1, the container agent (AWS Distro for OpenTelemetry) forwards container metrics to Amazon CloudWatch. In step 2, Amazon DevOps Guru is continually consuming those metrics and performing anomaly detection. If an anomaly is detected, then this generates an Insight, thereby triggering Amazon SNS notification as shown in step 3. In step 4, the operators access Amazon DevOps Guru console to examine the insight. Then, the operators can leverage the new user interface capability displaying which cluster, namespace, and pod/service is impacted along with correlated Amazon EKS metric(s).

New EKS Container Metrics in DevOps Guru
As part of the release, the following pod and node metrics are now tracked by DevOps Guru:

pod_number_of_container_restarts – number of times that a pod is restarted (e.g., image pull issues, container failure).
pod_memory_utilization_over_pod_limit – memory that exceeds the pod limit called out in resource memory limits.
pod_cpu_utilization_over_pod_limit – CPU shares that exceed the pod limit called out in resource CPU limits.
pod_cpu_utilization – percent CPU Utilization within an active pod.
pod_memory_utilization – percent memory utilization within an active pod.
node_network_total_bytes – total bytes over the network interface for the managed node (e.g., EC2 instance)
node_filesystem_utilization – percent file system utilization for the managed node (e.g., EC2 instance).
node_cpu_utilization – percent CPU Utilization within a managed node (e.g., EC2 instance).
node_memory_utilization – percent memory utilization within a managed node (e.g., EC2 instance).
Operator Scenario
The Kubernetes Operator is informed of an insight via Amazon SNS. The Amazon SNS message content appears in the following code, showing the originator and information identifying the InsightDescription, InsightSeverity, name of the container metric, and the Pod / EKS Cluster:

{
 "AccountId": "XXXXXXX",
 "Region": "<REGION>",
 "MessageType": "NEW_INSIGHT",
 "InsightId": "ADFl69Pwq1Aa6M373DhU0zkAAAAAAAAABuZzSBHxeiNexxnLYD7Lhb0vuwY9hLtz",
 "InsightUrl": "https://<REGION>.console.aws.amazon.com/devops-guru/#/insight/reactive/ADFl69Pwq1Aa6M373DhU0zkAAAAAAAAABuZzSBHxeiNexxnLYD7Lhb0vuwY9hLtz",
 "InsightType": "REACTIVE",
 "InsightDescription": "ContainerInsights pod_number_of_container_restarts Anomalous In Stack eksctl-devopsguru-eks-cluster-cluster",
 "InsightSeverity": "high",
 "StartTime": 1636147920000,
 "Anomalies": [
 {
 "Id": "ALAGy5sIITl9e6i66eo6rKQAAAF88gInwEVT2WRSTV5wSTP8KWDzeCYALulFupOQ",
 "StartTime": 1636147800000,
 "SourceDetails": [
 {
 "DataSource": "CW_METRICS",
 "DataIdentifiers": {
 "name": "pod_number_of_container_restarts",
 "namespace": "ContainerInsights",
 "period": "60",
 "stat": "Average",
 "unit": "None",
 "dimensions": "{\"PodName\":\"java-sample-app\",\"ClusterName\":\"devopsguru-eks-cluster\",\"Namespace\":\"aws-otel-eks\"}"
 }
 ....
 "awsInsightSource": "aws.devopsguru"
}
Amazon DevOps Guru Console collects the insights under the Insights selection as shown in the following figure. Select Insights to view the details.

Amazon DevOps Guru Insights. An insight is displayed with a status of Ongoing and Severity of High.

Aggregated Metrics provides the identification of the EKS Container Metrics that have errored. In this case, pod_memory_utilization_over_pod_limit and pod_number_of_container_restarts.

Aggregated Metrics panel with pod_memory_utilization_over_pod_limit and pod_number_of_container_restarts for the Amazon EKS cluster names devopsguru-eks-cluster. Graphically a timeline including time and date is displayed conveying the length of the anomaly.

Further details can be identified by selecting and expanding each insight as shown in the following figure.

Displays the ability to expand the cluster metrics providing further information on the PodName, Namespace and ClusterName. Furthermore, a search bar is provided to search on name, stack or service name.

Note that the display provides information around the Cluster, PodName, and Namespace. This helps operators maintaining large numbers of EKS Clusters to quickly isolate the offending Pod, its operating Namespace, and EKS Cluster to which it belongs. A search bar provides further filtering to isolate the name, stack, or service name displayed.

Cleaning Up
Follow the steps to delete the resources to prevent additional charges being posted to your account.

Amazon EKS Cluster Cleanup
Follow these steps to detach the customer managed policy and delete the cluster.

Detach customer managed policy, AWSDistroOpenTelemetryPolicy, via IAM Console.
Delete cluster using eksctl.
$ eksctl delete cluster devopsguru-eks-cluster --region <region>
2021-10-13 14:08:28 [i] eksctl version 0.69.0
2021-10-13 14:08:28 [i] using region <region>
2021-10-13 14:08:28 [i] deleting EKS cluster "devopsguru-eks-cluster"
2021-10-13 14:08:30 [i] will drain 0 unmanaged nodegroup(s) in cluster "devopsguru-eks-cluster"
2021-10-13 14:08:32 [i] deleted 0 Fargate profile(s)
2021-10-13 14:08:33 [✔] kubeconfig has been updated
2021-10-13 14:08:33 [i] cleaning up AWS load balancers created by Kubernetes objects of Kind Service or Ingress
2021-10-13 14:09:02 [i] 2 sequential tasks: { delete nodegroup "managed-ng-private", delete cluster control plane "devopsguru-eks-cluster" [async] }
2021-10-13 14:09:02 [i] will delete stack "eksctl-devopsguru-eks-cluster-nodegroup-managed-ng-private"
2021-10-13 14:09:02 [i] waiting for stack "eksctl-devopsguru-eks-cluster-nodegroup-managed-ng-private" to get deleted
2021-10-13 14:12:30 [i] will delete stack "eksctl-devopsguru-eks-cluster-cluster"
2021-10-13 14:12:30 [✔] all cluster resources were deleted
Conclusion
In the previous scenarios, demonstration of the new cluster organization and additional container metrics was performed. Both of these features further simplify and expand the ability for an operator to more easily identify issues within a container cluster when Amazon DevOps Guru detects anomalies. You can start building your own solutions that employ Amazon CloudWatch Agent / AWS Distro for OpenTelemetry Agent and Amazon DevOps Guru by reading the documentation. This provides a conceptual overview and practical examples to help you understand the features provided by Amazon DevOps Guru and how to use them.