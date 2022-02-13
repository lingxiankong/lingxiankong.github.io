---
layout: post
title: Securely access cloud resources between AWS and GCP
category: 技术
---

Nowadays, there are more and more businesses that are using combination and integration of multiple public clouds to deploy the workloads, which can be purely for the purpose of redundancy and system backup, or it can incorporate different cloud vendors for different services. Here are some scenarios that companies are working with multiple cloud providers:

* Build a CI/CD system in the GCP account, but wants to use those pipelines to deploy and manage resources in the AWS account. In order for this command to succeed
* Containerized applications running on Google Kubernetes Engine (GKE) need to access data stored in Amazon S3 bucket, send a message to an Amazon SNS topic or convert text to speech with Amazon Polly.

One explicit requirement for such deployment model is that an application running in a cloud could be able to access other cloud resources. The challenge is how to manage credentials when the application in one cloud provider need to manage resources in the other. The easiest way isn’t always the best.

## The Insecure Way

Let's talk about some examples.

In order to access Google Cloud resources outside of GCP, we need to prepare a GCP service account key file. Similarly, we need to export AWS Access Key and Secret Key for the AWS IAM User, and inject AWS credentials into the applications running in other clouds, so that the application can talk to AWS, e.g. either upload files to S3 or send messages to AWS SNS.

The problem is the credentials information needs to be moved between different environments or stored in some secret management system (e.g. Hashicorp Vault or Key Management Service in the cloud) in which case credentials for those services also need to be managed. The approach of distributing and saving cloud provider secrets is not the most secure approach.

What's more, using the credentials directly could also cause other problems: unintentionally committing the credentials into a GitHub repository; exposing those credentials via the unsafe RBAC settings in other systems; reusing credentials for different services and applications, etc.

## The Goal

Based on the problems described above, what we expect is a more secure way to config the cloud credentials for the applications running in a different cloud, ideally can be managed or configured outside of the workloads, not only because it’s convenient for users but also it improves organizational security by reducing account and password proliferation.

## Secure Way: AWS Workloads to Access GCP

Traditionally, accessing Google Cloud resources outside of GCP requires [service account key](https://cloud.google.com/iam/docs/service-accounts#service_account_keys). Suppose we have an application running in an AWS EC2 instance, what we expect is to make some configuration outside of the instance, so that the application is able to access GCP resources without storing and managing the service account key.

That's where GCP workload identity federation comes to help.

Using GCP workload identity federation allows external identity providers (AWS in this case) to impersonate a service account and access resources on Google Cloud, without explicitly copying a service account key from outside the EC2 instance. Google Cloud provides a [detailed document](https://cloud.google.com/iam/docs/workload-identity-federation) for this feature.

Behind the scene, you provide an AWS credential to the Google Cloud [Security Token Service](https://cloud.google.com/iam/docs/reference/sts/rest) (STS), which inturn verifies the identity on the credential, and then returns a federated access token in exchange.

What that basically states is, inside the EC2 instance, the GCP client library looks for AWS federation credentials from the EC2 metadata server or from the AWS env variables in the shell. Once the AWS credentials are acquired, the client library will perform the STS exchange and enable the GCP client library access.

The following steps will create a new EC2 instance and config to allow applications inside the instance to access Cloud Storage service.

1. You have an account with appropriate roles on a GCP project and enable related service APIs in GCP, especially the Security Token Service API. The roles should include: Workload Identity Pool Admin, Service Account Admin.

2. You have an AWS account. Get the AWS account ID and create an AWS role that is used to impersonate a service account on GCP.

   ```shell
   aws_account=$(aws sts get-caller-identity --output json | jq -r '.Account')

   cat <<EOF > ec2-role-trust-policy.json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
             "Effect": "Allow",
             "Principal": { "AWS" : "*" },
             "Action": "sts:AssumeRole"
           }
       ]
   }
   EOF

   aws_role="access-gcp-role"

   aws iam create-role \
       --role-name $aws_role \
       --assume-role-policy-document file://ec2-role-trust-policy.json
   ```

3. Create an instance profile using the new role and create an EC2 instance using the instance profile. Set values for the other variables based on your own AWS account.

   ```shell
   vm_profile="new-profile"
   vm_image=
   vm_type=
   vm_key=
   vm_sg=
   vm_subnet=

   aws iam create-instance-profile --instance-profile-name $vm_profile
   aws iam add-role-to-instance-profile --instance-profile-name $vm_profile --role-name $role

   aws ec2 run-instances --count 1 \
     --image-id $vm_image --instance-type $vm_type --key-name $vm_key \
     --security-group-ids $vm_sg --subnet-id $vm_subnet \
     --iam-instance-profile Name=$vm_profile
   ```

4. At the moment, applications inside the instance can't access GCP without providing a GCP service account key file. We are going to make that happen below.

5. In GCP, create a service account in the project with appropriate roles that you want to grant to the application in AWS EC2 instance. In this example, the service account is able to access Cloud Storage resources (`roles/storage.admin`).

   ```shell
   project=$(gcloud config get-value project 2>/dev/null)
   project_number=$(gcloud projects describe $project --format="value(projectNumber)")

   gcloud iam service-accounts create $serviceaccount_name --project $project --display-name $serviceaccount_name

   sa_email=$(gcloud iam service-accounts list --filter=displayName:"$serviceaccount_name" --format='value(email)')

   gcloud projects add-iam-policy-binding $project --member serviceAccount:$sa_email --role roles/storage.admin
   ```

6. In GCP, create workload identity pool and workload identity provider in the pool. Google recommends creating a new pool for each non-Google Cloud environment.

   ```shell
   pool_id="aws-pool"
   provider_id="aws-provider"

   gcloud iam workload-identity-pools create $pool_id \
       --location="global" \
       --description="AWS Identity Pool" \
       --display-name="AWS Identity Pool"

   gcloud iam workload-identity-pools providers create-aws $provider_id \
     --location="global"  \
     --workload-identity-pool=$pool_id \
     --account-id=$aws_account
   ```

7. In GCP, grant external identities permission to impersonate a service account, i.e. grant them the Workload Identity User role (`roles/iam.workloadIdentityUser`) on the service account. Here, we are giving the AWS accounts which have the role created above the permission to impersonate the service account created above as well.

   ```shell
   gcloud iam service-accounts add-iam-policy-binding $sa_email \
       --role=roles/iam.workloadIdentityUser \
       --member="principalSet://iam.googleapis.com/projects/${project_number}/locations/global/workloadIdentityPools/${pool_id}/attribute.aws_role/arn:aws:sts::${aws_account}:assumed-role/${aws_role}"
   ```

8. Now, all the configuration outside the EC2 instance have been finished. The configuration below is doing inside the EC2 instance.

   In the EC2 instance, after installing gcloud CLI, create a credential configuration file for GCP. Unlike a service account key file, a credential configuration file doesn't contain a private key and doesn't need to be kept confidential. Ensure all the variables are set before running the command.

   ```shell
   gcloud iam workload-identity-pools create-cred-config \
       projects/${project_number}/locations/global/workloadIdentityPools/${pool_id}/providers/${provider_id} \
       --service-account=${sa_email} \
       --aws \
       --output-file=$HOME/gcp-for-aws.json
   ```

9. With the GCP credential configuration file created above, Google Cloud SDK can do authentication automatically, so that the application can access the GCP Storage resources.

   ```shell
   gcloud auth login --cred-file=$HOME/gcp-for-aws.json --project $project
   ```

## Secure Way: GCP Workloads to Access AWS

Luckily, AWS provides a similar mechanism as GCP workload identity federation but with a different implementation, one of the significant differences is, there is code involved to get AWS credentials automatically.

AWS allows to create an IAM role for OpenID Connect Federation OIDC identity providers instead of IAM users. On the other hand, Google implements OIDC provider. In order to get temporary AWS credentials from AWS Security Token Service (STS), you need to provide a valid OIDC ID token.

Like the example above, we will make configurations to access the AWS resource for applications inside the GCP VM, again, without providing AWS access key and secret. We will follow the steps below.

1. In GCP, we have a service account that is associated with an existing VM or we could create a new one. We will skip the VM creation command for now and go ahead to get the service account unique client ID.

   ```shell
   client_id=$(gcloud iam service-accounts describe --format json $sa_email  | jq -r '.uniqueId')
   ```

2. In AWS, create a role for the GCP service account, attach a policy to the role, granting the S3 ready only access permission.

   ```shell
   cat <<EOF > gcp_trust_policy.json
   {
       "Version": "2012-10-17",
       "Statement": {
           "Sid": "RoleForGoogle",
           "Effect": "Allow",
           "Principal": {"Federated": "accounts.google.com"},
           "Action": "sts:AssumeRoleWithWebIdentity",
           "Condition": {"StringEquals": {"accounts.google.com:aud": "$client_id"}}
       }
   }
   EOF

   role=gcp-role

   aws iam create-role --role-name $role --assume-role-policy-document file://gcp_trust_policy.json --description "Allow GCP service account to access AWS resources"

   aws iam attach-role-policy --role-name $role --policy-arn "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"

   # Get the role arn that is used in the following step.
   role_arn=$(aws iam get-role --role-name $role --query Role.Arn --output text)
   ```

3. Now comes the tricky part. In the GCP VM, after installing AWS CLI, we prepare a python script which implements the function that getting temporary AWS credentials from AWS Security Token Service (STS) by providing a valid OIDC ID token, which in this case, the GCP service account token.

   The code requests an OIDC token from the local VM’s metadata endpoint, then calls the AWS `AssumeRoleWithWebIdentity` API using that token. The resulting AWS credentials are written into a standard location, at `$HOME/.aws/credentials` after which they can be used automatically by the AWS CLI.

   ```shell
   sudo curl -SL https://gist.github.com/lingxiankong/291f99f416caf1c1d40a748ecc7aa6c4/raw -o /usr/local/bin/get_aws_cred
   sudo chmod u+x /usr/local/bin/get_aws_cred

   mkdir ~/.aws
   cat <<EOF > ~/.aws/credentials
   [default]
   credential_process = /usr/local/bin/get_aws_cred $role_arn
   EOF
   ```

4. Finally, we have a GCP VM set up with time-limited (by default, 3600 seconds) credentials for the AWS IAM Role, and now we are able to list AWS S3 buckets or objects, remember we have `arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess` in step 2?

Obviously, the configuration steps to access AWS from GCP are less than that to access GCP from AWS and seems to be more straightforward, because most of the complexities are hiden in the code.

## Put Them Together

Well, manually setting up everything just for testing is tedious and error-prone, to make the process easier, I've created ansible playbooks that you can find [here](https://github.com/lingxiankong/ansible-cross-cloud-access-demo).

## Wrap up

This blog only aims at configuring resource access between AWS and GCP, but the process should be similar for others like Azure.

Ideally, there could be an SSO (Single Sign On) solution out there centrally that can be trusted to manage identities for applications deployed in different cloud providers. As the end user, we may need to register our cloud credentials in that central service and only use that service to delegate the access of the resources on all the clouds for our applications.
