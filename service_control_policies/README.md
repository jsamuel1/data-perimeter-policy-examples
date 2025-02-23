# Service control policy (SCP) examples

Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved. SPDX-License-Identifier: CC-BY-SA-4.0

## Table of Contents

* [Introduction](#introduction)
* [Description](#description)
* [Included data access patterns](#included-data-access-patterns)

## Introduction

SCPs are a type of [AWS Organizations](https://aws.amazon.com/organizations/) organization policy that you can use to establish the maximum access that can be delegated by account administrators to principals within your organization. SCPs allow you to enforce security invariants such as ensuring that your identities can access only company-approved data stores and that your corporate credentials can be used only from company-managed networks.

## Description

This folder contains examples of SCPs that enforce resource and network perimeter controls. These examples do not represent a complete list of valid data access patterns, and they are intended for you to tailor and extend them to suit the needs of your environment. 

For your network perimeter, this folder has examples of policies for enforcing controls on specific service roles and IAM principals tagged with the `data-perimeter-include` tag set to `true`. Some AWS services use [service roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_terms-and-concepts.html#iam-term-service-role) to perform tasks on your behalf. Some service roles are designed to be used by a service to directly call other services on your behalf as well as to make API calls from your code (for example, an [AWS Lambda](https://aws.amazon.com/lambda/) function role is used to publish logs to [Amazon CloudWatch](https://aws.amazon.com/cloudwatch/) and to make calls to AWS APIs from the Lambda function code). Because these services allow code execution, it is possible for a user to obtain the credentials associated with a service role. Therefore, you may want to enforce the use of such credentials from expected networks only. This folder provides examples for how to achieve this with [Amazon Elastic Compute Cloud (Amazon EC2)](https://aws.amazon.com/ec2/). Policy examples in this folder do not enforce network perimeter controls on any other IAM principals.

Use the following example SCPs individually or in combination:

* [resource_perimeter_policy](resource_perimeter_policy.json) – Enforces resource perimeter controls on all principals within your Organizations organization.
* [network_perimeter_policy](network_perimeter_policy.json) – Enforces network perimeter controls on IAM principals tagged with the `data-perimeter-include` tag set to `true`.
* [network_perimeter_ec2_policy](network_perimeter_ec2_policy.json) – Enforces network perimeter controls on service roles used by Amazon EC2 instances.
* [data_perimeter_governance_policy_1](data_perimeter_governance_policy_1.json) and [data_perimeter_governance_policy_2](data_perimeter_governance_policy_2.json) – Include statements to secure tags that are used for authorization controls. These SCPs also include statements that should be included in your data perimeter to account for specific data access patterns that are not covered by primary data perimeter controls. 

Note that the SCP examples in this repository use a [deny list strategy](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_strategies.html), which means that you also need a FullAWSAccess policy or other policy attached to your AWS Organizations organization entities to allow actions. You still also need to grant appropriate permissions to your principals by using identity-based or resource-based policies.

## Included data access patterns

The following policy statements are included in the SCP examples, each statement representing specific data access patterns.

### "Sid":"EnforceResourcePerimeterAWSResources"

This policy statement is included in the [resource_perimeter_policy](resource_perimeter_policy.json) and limits access to trusted resources that include AWS owned resources:

* Resources that belong to your Organizations organization specified by the organization ID (`<my-org-id>`) in the policy statement.
* Resources owned by AWS services. To permit access to AWS owned resources through the resource perimeter, two methods are used:
    * Relevant service actions are listed in the `NotAction` element of the policy. Actions on resources that allow cross-account access are further restricted in other statements of the policy (`"Sid":"EnforceResourcePerimeterAWSResourcesS3"`, `"EnforceResourcePerimeterAWSResourcesECR"`). 
        * `ssm:Describe*`, `ssm:List*`, `ssm:Get*` - Required for [AWS Systems Manager](https://aws.amazon.com/systems-manager/) global parameters.* *Some AWS services publish information about common artifacts as Systems Manager public parameters. For example, Amazon EC2 publishes information about Amazon Machine Images (AMIs) as public parameters. AWS automation or custom applications may need to access Systems Manager public parameters to support their operations. 
        * `iam:GetPolicy`, `iam:GetPolicyVersion`, `iam:ListEntitiesForPolicy`, `iam:ListPolicyVersions`, `iam:GenerateServiceLastAccessedDetails` - Required for AWS managed policies. AWS managed policies are owned by an AWS service account. 
        * `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage` - Required for [Amazon Elastic Kubernetes Service (Amazon EKS)](https://aws.amazon.com/eks/) add-ons, [GuardDuty EKS Runtime Monitoring](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty-eks-runtime-monitoring.html), and [Amazon SageMaker pre-built Docker images](https://docs.aws.amazon.com/sagemaker/latest/dg-ecr-paths/sagemaker-algo-docker-registry-paths.html).
    * `ec2:Owner` condition key:
        * Key value set to `amazon` - Required for your users and applications to be able to perform operations against public images that are owned by Amazon or a [verified partner](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/sharing-amis.html#verified-ami-provider) (for example, copying or launching instances using these images).
* Trusted resources that belong to an account outside of your Organizations organization. To permit access to a resource owned by an external account through the resource perimeter, relevant service actions have to be listed in the `NotAction` element of this statement (`<action>`). These actions are further restricted in the `"Sid":"EnforceResourcePerimeterThirdPartyResources"`. 

### "Sid":"EnforceResourcePerimeterAWSResourcesS3"

This policy statement is included in the [resource_perimeter_policy](resource_perimeter_policy.json) and limits access to trusted [Amazon Simple Storage Service (Amazon S3)](https://aws.amazon.com/s3/) resources:

* Amazon S3 resources that belong to your Organizations organization as specified by the organization ID (`<my-org-id>`) in the policy statement.
* Amazon S3 resources owned by AWS services that use your credentials to access those resources on your behalf. This access pattern applies when you access data via an AWS service and that service takes subsequent actions on your behalf by using your [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/) credentials. To account for this access pattern, the `s3:GetObject`, `s3:PutObject`, and `s3:PutObjectAcl` actions are first listed in the `NotAction` element of the `"Sid":"EnforceResourcePerimeterAWSResources"` statement. `"Sid":"EnforceResourcePerimeterAWSResourcesS3"` then uses the `aws:CalledVia` condition key to restrict these actions to AWS services only.

Example data access patterns:

* *AWS Data Exchange publishing and subscribing. *When importing assets from Amazon S3 to [AWS Data Exchange](https://aws.amazon.com/data-exchange/) ([publishing](https://docs.aws.amazon.com/data-exchange/latest/userguide/providing-data-sets.html)), AWS Data Exchange writes to AWS Data Exchange Amazon S3 buckets. Similarly, when exporting assets from AWS Data Exchange to Amazon S3 ([subscribing](https://docs.aws.amazon.com/data-exchange/latest/userguide/subscribe-to-data-sets.html)), AWS Data Exchange reads from AWS Data Exchange Amazon S3 buckets. [AWS Data Exchange uses the credentials of the user performing import and export operations to sign calls to Amazon S3](https://docs.aws.amazon.com/data-exchange/latest/userguide/access-control.html#:~:text=Amazon%20S3%20permissions). 
* *AWS Service Catalog operations. *When you create [AWS Service Catalog](https://aws.amazon.com/servicecatalog) products, Service Catalog stores the CloudFormation template in an Amazon S3 bucket owned by the service account. When you provision the product, Service Catalog downloads the template from the bucket. Calls to Amazon S3 are signed by your credentials. 

### "Sid":"EnforceResourcePerimeterAWSResourcesECR"

This policy statement is included in the [resource_perimeter_policy](resource_perimeter_policy.json) and limits access to trusted [Amazon Elastic Container Registry (Amazon ECR)](https://aws.amazon.com/ecr/) resources:

* Amazon ECR repositories that belong to your Organizations organization as specified by the organization ID (`<my-org-id>`) in the policy statement.
* Amazon ECR repositories owned by [Amazon Elastic Kubernetes Service (Amazon EKS)](https://aws.amazon.com/eks/) that host the Docker container images for [Amazon EKS add-ons](https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html). These repositories are accessed by using the IAM roles of your Amazon EKS nodes and managed node groups. To account for this access pattern, the `ecr:GetDownloadUrlForLayer`and`ecr:BatchGetImage` are first listed in the `NotAction` element of the `"Sid":"EnforceResourcePerimeterAWSResources"` statement. `"Sid":"EnforceResourcePerimeterAWSResourcesECR"` then uses the `aws:ResourceAccount` condition key to restrict these actions to Amazon ECR repositories owned by the AWS service accounts. `<ecr-account-id>` can vary by AWS Region, and you might need to allow multiple account IDs if you are operating in multiple Regions. For a complete list of Amazon EKS managed AWS accounts that host Amazon ECR repositories, see the [Amazon EKS documentation about add-on images](https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html). The account ID is the first 12 digits of the registry URL.
* Amazon ECR repositories owned by [Amazon GuardDuty](https://aws.amazon.com/guardduty/) that host the Docker container images for [GuardDuty EKS Runtime Monitoring](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty-eks-runtime-monitoring.html). These repositories are accessed by using the IAM roles of your Amazon EKS nodes and managed node groups. To account for this access pattern, the `ecr:GetDownloadUrlForLayer`and`ecr:BatchGetImage` are first listed in the `NotAction` element of the `"Sid":"EnforceResourcePerimeterAWSResources"` statement. `"Sid":"EnforceResourcePerimeterAWSResourcesECR"` then uses the `aws:ResourceAccount` condition key to restrict these actions to Amazon ECR repositories owned by the AWS service accounts. `<ecr-account-id>` can vary by AWS Region, and you might need to allow multiple account IDs if you are operating in multiple Regions. For a complete list of Amazon GuardDuty managed AWS accounts that host Amazon ECR repositories, see the [GuardDuty EKS Runtime Monitoring documentation page](https://docs.aws.amazon.com/guardduty/latest/ug/eks-runtime-agent-ecr-image-uri.html). The account ID is the first 12 digits of the registry URL.
 *Amazon SageMaker ECR repositories hosting [Amazon SageMaker pre-built Docker images](https://docs.aws.amazon.com/sagemaker/latest/dg-ecr-paths/sagemaker-algo-docker-registry-paths.html). These repositories may be accessed by the IAM roles of your SageMaker notebooks or SageMaker studio notebooks. To account for this access pattern, the `ecr:GetDownloadUrlForLayer`and`ecr:BatchGetImage` are first listed in the `NotAction` element of the `"Sid":"EnforceResourcePerimeterAWSResources"` statement. `"Sid":"EnforceResourcePerimeterAWSResourcesECR"` then uses the `aws:ResourceAccount` condition key to restrict these actions to Amazon ECR repositories owned by the AWS service accounts. `<ecr-account-id>` can vary by AWS Region, and you might need to allow multiple account IDs if you are operating in multiple Regions. The AWS documentation has a complete list of [ECR repositories host SageMaker pre-built Docker images](https://docs.aws.amazon.com/sagemaker/latest/dg-ecr-paths/sagemaker-algo-docker-registry-paths.html)


### "Sid":"EnforceResourcePerimeterThirdPartyResources"

This policy statement is included in the [resource_perimeter_policy](resource_perimeter_policy.json) and limits access to trusted resources that include third party resources:

* Resources that belong to your Organizations organization and are specified by the organization ID (`<my-org-id>`) in the policy statement.
* Trusted resources that belong to an account outside of your Organizations organization are specified by account IDs of third parties (`<third-party-account-a>` and `<third-party-account-b>`) in the policy statement. Further restrict access by specifying allowed actions in the Action element of the policy statement. These actions also have to be listed in the `NotAction` element of `"Sid":"EnforceResourcePerimeterAWSResources"`.

### "Sid": "EnforceNetworkPerimeter"

This policy statement is included in the [network_perimeter_policy](network_perimeter_policy.json) and limits access to expected networks for IAM principals tagged with the `data-perimeter-include` tag set to `true. Expected networks are defined as follows:

* Your organization’s on-premises data centers and static egress points in AWS such as NAT gateways that are specified by IP ranges (`<my-corporate-cidr>`) in the policy statement. 
* Your organization’s VPCs specified by VPC IDs (`<my-vpc>`) in the policy statement.  
* Networks of AWS services that use your credentials to access resources on your behalf as denoted by `aws:ViaAWSService` in the policy statement. This access pattern applies when you access data via an AWS service and that service takes subsequent actions on your behalf by using your IAM identity credentials. 
* Networks of AWS services that use [service roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_terms-and-concepts.html) to access resources on your behalf. To permit access from AWS networks through the network perimeter, two methods are used:
    * Relevant service actions are listed in the NotAction element of the policy statement:
        * `dax:GetItem`, `dax:BatchGetItem`, `dax:Query`, `dax:Scan`, `dax:PutItem`, `dax:UpdateItem`, `dax:DeleteItem`, `dax:BatchWriteItem`, and `dax:ConditionCheckItem` – Required for [Amazon DynamoDB Accelerator (DAX)](https://aws.amazon.com/dynamodb/dax/) operations. At runtime, the DAX client directs all of your application's DynamoDB API requests to the DAX cluster, which is designed to run in your VPC; even though these API requests originate from your VPC, they do not traverse a VPC endpoint.
        * The `aws:PrincipalArn` condition key:
            * The key’s value is set to `arn:aws:iam::*:role/aws:ec2-infrastructure` – Required for Amazon EBS volume decryption.* *When you mount an encrypted Amazon EBS volume to an Amazon EC2 instance, Amazon EC2 calls AWS KMS to decrypt the data key that was used to encrypt the volume. The call to AWS KMS is signed by an IAM role, `arn:aws:iam::*:role/aws:ec2-infrastructure`, which is created in your account by Amazon EC2. 

Example data access patterns:

* *Athena query*. When an application running in your VPC [creates an Athena query](https://docs.aws.amazon.com/athena/latest/ug/getting-started.html), Athena uses the application’s credentials to make subsequent requests to Amazon S3 to read data from your bucket and return results. 
* *Elastic Beanstalk operations. *When a user [creates an application by using Elastic Beanstalk](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/applications.html), Elastic Beanstalk launches an environment and uses your IAM credentials to create and configure other AWS resources such as Amazon EC2 instances. 
* *Amazon S3 server-side encryption with AWS KMS (SSE-KMS)*. When you upload an object to an Amazon S3 bucket with the default [SSE-KMS encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingKMSEncryption.html) enabled, Amazon S3 makes a subsequent request to AWS KMS in order to generate a data key from your CMK to encrypt the object. The call to AWS KMS is signed by using your credentials. A similar pattern is observed when a user tries to download an encrypted object when Amazon S3 calls AWS KMS to decrypt the key that was used to encrypt the object. Other services such as Secrets Manager use a similar integration with AWS KMS.
* *AWS Data Exchange publishing and subscribing* (described in  `"Sid":"EnforceResourcePerimeterAWSResourcesS3"` earlier in this document).
* *AWS Service Catalog operations* (described in `"Sid":"EnforceResourcePerimeterAWSResourcesS3"` earlier in this document).


Additionally, `es:ES*` is included in the `NotAction` element of the policy statement, which is required to use HTTP methods on [Amazon OpenSearch Service](https://aws.amazon.com/opensearch-service/the-elk-stack/what-is-opensearch/) domains that reside within a VPC for which security groups are the mechanism to enforce IP-based access policies.

### "Sid": "EnforceNetworkPerimeterOnEC2Roles"

This policy statement is included in the [network_perimeter_ec2_policy](network_perimeter_ec2_policy.json) and limits access to expected networks for service roles used by Amazon EC2 instance profiles. Expected networks are defined as follows:
* Your on-premises data centers and static egress points in AWS such as a NAT gateway that are specified by IP ranges (`<my-corporate-cidr>`) in the policy statement. 
* Your organization’s VPCs that are specified by VPC IDs (`<my-vpc>`) in the policy statement.  
* Networks of AWS services that use your credentials to access resources on your behalf as denoted by `aws:ViaAWSService` in the policy statement. This access pattern applies when you access data via an AWS service, and that service takes subsequent actions on your behalf by using your IAM credentials. 
* Networks of AWS services that use [service roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_terms-and-concepts.html) to access resources on your behalf. To permit access from AWS networks through the network perimeter, two methods are used:
    * Relevant service actions are listed in the NotAction element of the policy statement:
        * `dax:GetItem`, `dax:BatchGetItem`, `dax:Query`, `dax:Scan`, `dax:PutItem`, `dax:UpdateItem`, `dax:DeleteItem`, `dax:BatchWriteItem`, and `dax:ConditionCheckItem` – Required for [Amazon DynamoDB Accelerator (DAX)](https://aws.amazon.com/dynamodb/dax/) operations. At runtime, the DAX client directs all of your application's DynamoDB API requests to the DAX cluster, which is designed to run in your VPC; even though these API requests originate from your VPC, they do not traverse a VPC endpoint.
    * The `aws:PrincipalArn` condition key:
        * The key’s value is set to `arn:aws:iam::*:role/aws:ec2-infrastructure` – Required for Amazon EBS volume decryption.* *When you mount an encrypted Amazon EBS volume to an Amazon EC2 instance, Amazon EC2 calls AWS KMS to decrypt the data key that was used to encrypt the volume. The call to AWS KMS is signed by an IAM role, `arn:aws:iam::*:role/aws:ec2-infrastructure`, which is created in your account by Amazon EC2. 

Additionally, `es:ES*` is included in the `NotAction` element of the policy statement, which is required to use HTTP methods on Amazon OpenSearch Service domains that reside in a VPC for which security groups are the mechanism to enforce IP-based access policies.

The `ec2:SourceInstanceARN` condition key is used to target role sessions that are created for applications running on your Amazon EC2 instances. 

### "Sid": "PreventRAMExternalResourceShare"

This statement is included in the [data_perimeter_governance_policy_1](data_perimeter_governance_policy_1.json) and denies the creation of or updates to [AWS Resource Access Manager (AWS RAM)](https://aws.amazon.com/ram/) resource shares that allow sharing with external principals.

[Some AWS resources](https://docs.aws.amazon.com/ram/latest/userguide/shareable.html#shareable-r53.) allow cross-account sharing via AWS RAM instead of resource-based policies. By default, AWS RAM shares allow sharing outside of an Organizations organization. You can explicitly [restrict sharing of resources outside of AWS Organizations](https://docs.aws.amazon.com/ram/latest/userguide/working-with-sharing-create.html) and then limit AWS RAM actions based on this configuration.

### "Sid": "PreventExternalResourceShare"

This statement is included in the [data_perimeter_governance_policy_1](data_perimeter_governance_policy_1.json) and restricts resource sharing by capabilities that are embedded into services.

Some AWS services use neither resource-based policies nor AWS RAM.

Example data access patterns:

* [Amazon EC2 AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/sharing-amis.html): You can share AMIs with other accounts or make them public with the `ModifyImageAttribute` and `ModifyFPGAImageAttribute` APIs. 
* [Amazon EBS snapshots](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-modifying-snapshot-permissions.html): You can share Amazon EBS snapshots with other accounts, or you can make them public with the `ModifySnapshotAttribute` API.
* [VPC endpoint connections](https://docs.aws.amazon.com/vpc/latest/privatelink/configure-endpoint-service.html): You can grant permissions to another account to connect to your VPC endpoint service with the `ModifyVpcEndpointServicePermissions` API.
* [Systems Manager documents (SSM documents)](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-ssm-docs.html): You can share SSM documents with other accounts or make them public with the `ModifyDocumentPermission` API.
* [Amazon RDS Snapshots](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ShareSnapshot.html): You can share RDS and RDS cluster snapshots with other accounts or make them public with the `ModifyDBSnapshotAttribute` and `ModifyDBClusterSnapshotAttribute` APIs.
* [Amazon Redshift datashare](https://docs.aws.amazon.com/redshift/latest/dg/authorize-datashare-console.html): You can authorize the sharing of a datashare with other accounts with the `AuthorizeDataShare` API. You can also share a snapshot with other accounts with `AuthorizeSnapshotAccess` API.
* [AWS Directory Service directory](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_directory_sharing.html): You can share a directory with other accounts with the `ShareDirectory` API.
* [Amazon CloudWatch Logs subscription filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters.html): You can send CloudWatch Logs to cross-account destinations with the `PutSubscriptionFilter` API.
* [AWS Glue Data Catalog](https://docs.aws.amazon.com/lake-formation/latest/dg/granting-catalog-perms-TBAC.html) databases: You can grant data catalog permissions to another account by using the AWS Lake Formation tag-based access control method with the `GrantPermissions` and `BatchGrantPermissions` APIs.
* [Amazon AppStream 2.0 image](https://docs.aws.amazon.com/appstream2/latest/developerguide/administer-images.html#share-image-with-another-account): You can share an Amazon AppStream 2.0 image that you own with other accounts with the `UpdateImagePermissions` API.

### "Sid": "ProtectActionsNotSupportedByPrimaryDPControls"

This statement is included in [data_perimeter_governance_policy_1](data_perimeter_governance_policy_1.json) and restricts actions that are not supported by primary data perimeter controls, such as those listed in the[ResourceOrgID condition key page](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-resourceorgid). 

Example data access patterns:

* [Transit gateway peering connections](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-peering.html): You can create and manage a TGW peering connection with another account with the  `CreateTransitGatewayPeeringAttachment`,  `AcceptTransitGatewayPeeringAttachment`, `RejectTransitGatewayPeeringAttachment`, and `DeleteTransitGatewayPeeringAttachment` APIs. 
* [VPC peering connections](https://docs.aws.amazon.com/vpc/latest/peering/create-vpc-peering-connection.html): You can create and manage a VPC peering connection with another account with the `CreateVpcPeeringConnection`,  `AcceptVpcPeeringConnection`, `RejectVpcPeeringConnection`, and `DeleteVpcPeeringConnection` APIs.
* [VPC endpoint connections](https://docs.aws.amazon.com/vpc/latest/privatelink/configure-endpoint-service.html): You can create and manage an endpoint service connection with another account with the `CreateVpcEndpoint`, `AcceptVpcEndpointConnections`, and   `RejectVpcEndpointConnections` APIs. 
* [Amazon EC2 AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/sharing-amis.html): You can copy an image shared from a different account with the `CopyImage` and `CopyFpgaImage` APIs.
* [Amazon EBS snapshots](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-modifying-snapshot-permissions.html): You can copy a snapshot shared from a different account with the `CopySnapshot` API.
* [Amazon EBS volumes](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-creating-volume.html): You can create a volume from a snapshot shared from a different account with the `CreateVolume` API.
* [Amazon Route 53 private hosted zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zone-private-associate-vpcs-different-accounts.html): You can associate and manage a VPC with a private hosted zone in a different account with the `CreateVPCAssociationAuthorization`,  `AssociateVPCWithHostedZone`, `DisassociateVPCFromHostedZone`, `ListHostedZonesByVPC`, and `DeleteVPCAssociationAuthorization`  APIs.

You can also consider using service-specific condition keys such as `ec2:AccepterVpc` and `ec2:RequesterVpc` to restrict actions that are not supported by primary data perimeter controls (See [Work within a specific account](https://docs.aws.amazon.com/vpc/latest/peering/security-iam.html#vpc-peering-iam-account)).

### "Sid": "ProtectDataPerimeterTags"

This statement is included in the [data_perimeter_governance_policy_1](data_perimeter_governance_policy_1.json) and prevents the attaching, detaching, and modifying of tags used for authorization controls within the data perimeter.

### "Sid": "PreventS3PublicAccessBlockConfigurations"

This statement is included in the [data_perimeter_governance_policy_1](data_perimeter_governance_policy_1.json) and prevents users from altering S3 Block Public Access configurations.

[S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html) provides settings for access points, buckets, and accounts to help you manage public access to Amazon S3 resources. With S3 Block Public Access, account administrators and bucket owners can set up centralized controls to limit public access to their Amazon S3 resources that are enforced, regardless of how the resources are created. 


### "Sid": "PreventPublicBucketACL"

This statement is included in the [data_perimeter_governance_policy_1](data_perimeter_governance_policy_1.json) and prevents users from applying public read and public read-write canned access control lists to Amazon S3 buckets.


### "Sid":"PreventLambdaFunctionURLAuthNone"

This statement is included in the [data_perimeter_governance_policy_2](data_perimeter_governance_policy_2.json) and denies the creation of Lambda functions that have `lambda:FunctionUrlAuthType` set to `NONE`.

[Lambda function URLs](https://docs.aws.amazon.com/lambda/latest/dg/lambda-urls.html) is a feature that lets you add HTTPS endpoints to your Lambda functions. When configuring a function URL for a new or existing function, you can set the `AuthType` parameter to `NONE`, which means Lambda won’t check for any IAM SigV4 signatures before invoking the function. If a function’s resource-based policy explicitly allows for public access, the function is open to unauthenticated requests.

###  "Sid": "PreventDeploymentCodeStarConnectionsCloudShell"

This statement  is included in the [data_perimeter_governance_policy_2](data_perimeter_governance_policy_2.json) and limits the use of [AWS CloudShell](https://aws.amazon.com/cloudshell/) and [AWS CodeStar Connections](https://docs.aws.amazon.com/codestar-connections/latest/APIReference/Welcome.html).

AWS services such as CloudShell and AWS CodeStar Connections do not support deployment within a VPC and provide direct access to the internet that is not controlled by your VPC. You can block the use of such services by using SCPs or implementing your own proxy solution to inspect egress traffic.

### "Sid":"PreventNonVPCDeploymentSageMaker", "Sid":"PreventNonVPCDeploymentGlueJob", and "Sid":"PreventNonVPCDeploymentLambda"

These statements are included in the [data_perimeter_governance_policy_2](data_perimeter_governance_policy_2.json) and explicitly deny relevant [Amazon SageMaker](https://aws.amazon.com/sagemaker/), [AWS Glue](https://aws.amazon.com/glue/) and [AWS Lambda](https://aws.amazon.com/lambda/) operations unless they have VPC configurations specified in the requests. Use these statements to enforce deployment in a VPC for these services.

Services such as Lambda, AWS Glue, and SageMaker support different deployment models. For example, [Amazon SageMaker Studio](https://aws.amazon.com/pm/sagemaker/) and [SageMaker notebook instances](https://docs.aws.amazon.com/sagemaker/latest/dg/nbi.html) allow direct internet access by default. However, they provide you with the capability to configure them to run within your VPC so that you can inspect requests by using VPC endpoint policies (against identity and resource perimeter controls) and enforce the network perimeter.




