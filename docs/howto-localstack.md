# How to work with LocalStack

This document describes how to cold start the Cloud Posse Ref Arch on LocalStack. As of writing,
the paid version of LocalStack, i.e., `LocalStack Pro` is needed.

## Describe Your Environment
Whenever these appear in this guide, use your own values:
* `<Root Account Number> = 123456789000`
* `<Repo Name> = infra`
* `<LocalStack Auth Token> = XXXXXXXXXX...`

## One Time Setup
### Create a Copy of the Repo
Make a copy of the Cloud Posse Ref Arch repo, and use this guide over there.

### Set Root Account Number
In the `stacks` dir, search and replace `__ROOT_ACCOUNT_NUMBER__` with your `<Root Account Number>`:
```
find stacks -name '*.yaml' -exec sed --in-place -e 's/__ROOT_ACCOUNT_NUMBER__/<Root Account Number>/g' {} \;
```

### Install LocalStack
Install LocalStack with `brew install localstack`.

### Update Geodesic
Update Geodesic to use the latest versions and install the prerequisites for LocalStack.

Edit the file `components/docker/<Repo Name>/Dockerfile`:
```
ARG GEODESIC_VERSION=4.0.1
ARG GEODESIC_OS=debian
ARG ATMOS_VERSION=1.160.4
ARG TOFU_VERSION=1.9.0
...
...
...

RUN apt-get install -y docker.io \
      && pip install terraform-local awscli-local

WORKDIR /
```

Rebuild Geodesic with `make all`.

### Configure the AWSUTILS Provider
As of writing, the EC2 endpoint in cloudposse/awsutils provider cannot be set via environment variable
(See [here](https://github.com/cloudposse/terraform-provider-awsutils/pull/75) for more details).
Therefor, the following workaround is needed:

* Create the file `stacks/mixins/localstack/awsutils.yaml` with the following content:
```
terraform:
  providers:
    awsutils:
      endpoints:
        ec2: "http://localhost.localstack.cloud:4566"
```

* Add the following `import` to `stacks/catalog/tfstate-backend.yaml`:
```
import:
  - mixins/localstack/awsutils
```

### Adjustments for LocalStack
* In `stacks/catalog/account-map.yaml`, set `root_account_aws_name: master`. See [here](https://github.com/orgs/cloudposse/discussions/40) for more details.
* Budgets are not yet supported in LocalStack. In `stacks/catalog/account-settings.yaml` set `budgets_enabled: false`.

## Working with LocalStack
### Start LocalStack
Start LocalStack in a new terminal window:
```
export LOCALSTACK_AUTH_TOKEN="<LocalStack Auth Token>"
export S3_ENDPOINT_URL=http://s3.localhost.localstack.cloud:4566
localstack start
```

### Start Leapp
In Leapp, start a LocalStack session, with the profile `default`.

### Start Geodesic
Start Geodesic with docker support, passing `--with-docker` to the startup script.

#### Clean State
To start fresh it's best to clean the previous state (if any):
```
atmos tofu clean --yes
find /workspace/components -name '*.json' -exec rm {} \;
```

#### Prepare for LocalStack
Run the following to prepare this Geodesic shell for LocalStack:

```
unset AWS_PROFILE

# For Hashi/aws provider
export AWS_ENDPOINT_URL=http://localhost.localstack.cloud:4566
export AWS_ENDPOINT_URL_S3=http://s3.localhost.localstack.cloud:4566

# For Cloud Posse/awsutils provider
export TF_AWS_STS_ENDPOINT=http://localhost.localstack.cloud:4566
export TF_AWS_EC2_ENDPOINT=http://localhost.localstack.cloud:4566 # No effect before PR-75 is merged
export TF_AWS_S3_ENDPOINT=http://s3.localhost.localstack.cloud:4566
export TF_AWS_DYNAMODB_ENDPOINT=http://localhost.localstack.cloud:4566

# Add the LocalStack container's IP as first Nameserver
localstack_container_ip=$(docker inspect localstack-main | jq -r '.[0].NetworkSettings.Networks[].IPAddress')
sed -e 's/nameserver \(.*\)/nameserver '"$localstack_container_ip"'\nnameserver \1/' /etc/resolv.conf >/tmp/resolv.conf && cat /tmp/resolv.conf >/etc/resolv.conf
```

## Cold Start the Ref Arch
The following is carried in the Geodesic shell configured in the previous step.

### Supress Git Warnings
Run the following to supress git warnings:
```
git config --global --add safe.directory /workspace/components/terraform/account-settings/.terraform/modules/service_quotas
git config --global --add safe.directory /workspace/components/terraform/cloudtrail-bucket/.terraform/modules/cloudtrail_s3_bucket.s3_access_log_bucket.aws_s3_bucket.s3_user.s3_user
git config --global --add safe.directory /workspace/components/terraform/account-settings/.terraform/modules/budgets.kms_key
```

### Create SuperAdmin
```
export AWS_ACCESS_KEY_ID='<Root Account Number>'
export AWS_SECRET_ACCESS_KEY=test
aws iam create-user --user-name SuperAdmin
aws iam create-access-key --user-name SuperAdmin >/tmp/superadmin_key
export AWS_ACCESS_KEY_ID=$(cat /tmp/superadmin_key | jq -r '.AccessKey.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(cat /tmp/superadmin_key | jq -r '.AccessKey.SecretAccessKey')
```
Now run `aws sts get-caller-identity` and verify that it's `SuperAdmin`.

### Vendoring
Start with the vendoring workflow:
```
atmos workflow vendor -f baseline
```

### Tweak account-map
Before continuing, there's some tweaking to do in the `account-map` component.

Add `ec2` endpoint to `components/terraform/account-map/modules/iam-roles/providers.tf`:
```
provider "awsutils" {
  ...
  ...
  ...

  endpoints {
    ec2 = "http://localhost.localstack.cloud:4566"
  }
}
```

### The Rest of the Workflow
Say a short prayer, cross your fingers, and then knock on wood three times.
```
atmos workflow init/tfstate -f baseline
atmos workflow deploy/organization -f accounts
atmos workflow deploy/accounts -f accounts
atmos workflow deploy/account-settings -f accounts
atmos workflow deploy -f baseline
```


## Notes
* Report issues in the Cloud Posse's [Discussions](https://github.com/orgs/cloudposse/discussions)
* What about Slack settings in `/workspace/stacks/catalog/account-settings.yaml` ?
* The cold start guide is [here](https://docs.cloudposse.com/layers/accounts/tutorials/manual-configuration/)
