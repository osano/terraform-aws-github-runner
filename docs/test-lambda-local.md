# Lambda - Test locally

This README provide guidance for testing the lambda locally / and or in AWS. This guide assumes you are familiar with AWS, lambda and Node. If not mention explicit comments provided should be executed from the root of the lambda package.

## Testing in AWS

Just navigate to AWS Lambda en trigger a test event. Provide an event that match the required input. For lambdas that does not require a specific event, just sent any event.


## Testing locally

Testing locally can be done in two ways. Using aws SAM framework, or run via a wrapper to simulate the event to invoke the lambda. Both setups requires that the required input environment variables are set, and AWS resources on which the lambda depends are available. We advise for testing the lambda locally to first create your own deployment to AWS, this will simplify the setup on depended AWS resources. For example based on the de [default example](../../../../examples/default/).

Locally test setups instruction are available for the following lambda's

- [runner-binary-syncer](./moduele/../../modules/runner-binaries-syncer/lambdas/runner-binaries-syncer) - This lambda does not need any input, no event is required. Supported via SAM and local Node.

### Extend deployment configuration

Add the code below to your Terraform deployment, to allow your principal to use the Lambda role, and retrieve the lambda configuration. Update your Terraform deployment and apply the changes.

```hcl
data "aws_caller_identity" "current" {}

module "runners" {
  
  ...

  # Assume you have a profile with Admin privileges, allow yo to switch to the Lambda role
  lambda_principals = [{
    type        = "AWS"
    identifiers = ["arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"]
  }]

}

output "development" {
  value = {
    lambda_syncer = module.runners.binaries_syncer.lambda
  }
}```

Once you have updated your Terraform deployment read the lambda configuration into your environment. Run the commands below in your Terraform workspace folder.

```bash
LAMBDA_ENV=$(terraform output -json development | jq -r '.lambda_syncer.environment[].variables' | jq -r "to_entries|map(\"\(.key)=\(.value|tostring)\")|.[]")
for x in $LAMBDA_ENV ; do echo setting $x; export $x; done
```

### Testing with SAM

This setup requires AWS SAM cli and Docker is installed. First update the AWS config (`~/.aws/config`) so you can use easy switch to the role used by the lambda.

```properties
[profile gh-development]
source_profile=<OPTIONAL_SOURCE_PROFILE>
region=<DEFAULT_REGION>
role_arn=<ARN_CHECK_TF_OUTPUT>
```

Now you can set the profile and region as environment variable or pass as argument to sam.

```
export AWS_REGION=<region>
export AWS_PROFILE=gh-development
```

For SAM a `template.yml` defines the lambda for running local. Thats all, now build your the lambda with `yarn run dist` and next invoke the lambda with `sam local invoke`.


### With Node

Alternatively of using SAM you can use Node with ts-node-dev to test the code locally. The drawback is that you have to setup AWS credentials into you shell. Also you are depending on a tiny wrapper (`local.ts`), and your local Node version.

The AWS SDK seems not to handle environment variables for profile, the only option to pass the role is via credentials. You can get credentials via STS for the role.

```bash

role=$(aws sts assume-role \
    --role-arn "<ROLE>" \
    --duration-seconds 3600 --role-session-name "dev")

export AWS_ACCESS_KEY_ID=$(echo $role | jq -r .Credentials.AccessKeyId)
export AWS_SECRET_ACCESS_KEY=$(echo $role | jq -r .Credentials.SecretAccessKey)
export AWS_SESSION_TOKEN=$(echo $role | jq -r .Credentials.SessionToken)
```

Next set the region `export AWS_REGION=<region>`. Now you can run the lambda locally via `yarn run watch`.