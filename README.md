# Compliance Framework version 2

## Key changes

1. Create Config Rules git repository and build the rules separately from aws config engine
2. Decouple IAM resources from the original Cloudformation template. IAM resources deployment required higher permissions
3. Add compliance-package-codebuild for github enterprise and codebuild integration and package rules and operational scripts as a single zip file
4. Create aggregator for config aggregated view in compliance account
5. Create and specified RDK lambda functions IAM role
6. Encrypted S3 buckets with created KMS key
7. Refactored and parameterized IAM resources names. And created env_var.properties for project specific variable.
8. Create application account config service role and specify RDK lambda functions to assume role in application account
9. Embedded account_list.json in rulesets-build/ for CI/CD. Parameter AccountList changed to AccountListLocalPath
10. Support new RDK remediation feature
11. Add script to automatically execute athena named queries
12. Implement RDK Lib lambda layer to all custom rules.
13. Add support for creating RDK lambda functions within VPC
14. Add support for config rule tagging

## Config files

1. [env_var.properties](./env_var.properties): This file is used for overriding variables used by the individual CloudFormation templates. The individual CloudFormation templates MUST NOT define parameters.
2. [account_list.json](rulesets-build/config/account_list.json): This file is used to maintain metadata of each account and the rulesets they will be using.
3. [rules_lambda_vpc.json](rulesets-build/config/rules_lambda_vpc.json): This file is used to provide the vpc detail if you want to create lambda functions within VPC.

## Deployment Steps

### Compliance Account Setup

1. (IAM admin) create iam resources for codePipeline, rdklambda, ETL Lambda

    ```bash
    aws cloudformation deploy \
        --stack-name compliance-iam \
        --template-file compliance-account-iam.yaml \
        --parameter-overrides $(cat env_var.properties) \
        --no-fail-on-empty-changeset \
        --capabilities CAPABILITY_NAMED_IAM \
        --region <mainRegion>
    ```

2. create kms key and grant permissions for automation iam roles

    ```bash
    aws cloudformation deploy \
        --stack-name compliance-kms \
        --template-file compliance-account-kms-setup.yaml \
        --parameter-overrides $(cat env_var.properties) \
        --no-fail-on-empty-changeset \
        --region <mainRegion>
    ```

3. create compliance framework CI/CD pipeline, S3 buckets, Config Aggregator, Firehose, log ETL lambda

    ```bash
    aws cloudformation deploy \
        --stack-name compliance \
        --template-file compliance-account-initial-setup.yaml \
        --parameter-overrides $(cat env_var.properties) \
        --no-fail-on-empty-changeset \
        --region <mainRegion>
    ```

### Application Account Setup

1. Add accountID in env_var.properties
2. Fill out [account_list.json](ruleset-build/account_list.json) and run steps 3 and 4 in the associated accounts.
    - Make sure config recorder and config service role is created. Otherwise, add DeployAWSConfig=true in env_var.properties
3. (IAM admin in appicaltion account) Create iam resources for compliance to assume and access application account

    ```bash
    aws cloudformation deploy \
        --stack-name application-iam \
        --template-file application-account-iam.yaml \
        --parameter-overrides $(cat env_var.properties) \
        --no-fail-on-empty-changeset \
        --capabilities CAPABILITY_NAMED_IAM \
        --region <mainRegion>
    ```

4. (**Appicaltion account - targetRegion) Create compliance framework config rule to check and trigger deployment

    ```bash
    aws cloudformation deploy \
        --stack-name application \
        --template-file application-account-initial-setup.yaml \
        --parameter-overrides $(cat env_var.properties) \
        --no-fail-on-empty-changeset \
        --region <targetRegion>
    ```

5. (**Compliance account - mainRegion) Update Config Aggregator for new account

    ```bash
    aws cloudformation deploy \
        --stack-name compliance \
        --template-file compliance-account-initial-setup.yaml \
        --parameter-overrides $(cat env_var.properties) \
        --no-fail-on-empty-changeset \
        --region <mainRegion>
    ```

6. Fill out [account_list.json](ruleset-build/account_list.json) with new account configurations
    - Make sure config recorder and config service role is created. Otherwise, add DeployAWSConfig=true in env_var.properties

7. Zip the 2 directories "rules/" and "rulesets-built/" into "ruleset.zip", including the directories themselves.
8. Copy the "ruleset.zip" in the source bucket (i.e. by default "compliance-engine-codebuild-source-account_id-region_name")
9. Go to CodePipeline, then locate the pipeline named "Compliance-Engine-Pipeline". Wait that it auto-triggers (it might show "Failed" when you check for the first time).

### Add new region
New region must be setup in compliance account before setting up in application account

1. Add OtherActiveRegions with comma delimited list of regions in [env_var.properties](./env_var.properties)
2. [**Compliance Account - mainRegion**] Run the following to update CodePipeline for new region

    ```bash
    aws cloudformation deploy \
        --stack-name compliance \
        --template-file compliance-account-initial-setup.yaml \
        --parameter-overrides $(cat env_var.properties) \
        --no-fail-on-empty-changeset \
        --region <mainRegion>
    ```

3. [**Compliance Account - newRegion**] Create KMS key for the region and grant permission to iam roles for automation in main region

    ```bash
    aws cloudformation deploy \
        --stack-name compliance-kms \
        --template-file compliance-account-kms-setup.yaml \
        --parameter-overrides $(cat env_var.properties) \
        --no-fail-on-empty-changeset \
        --region <newRegion>
    ```

4. [**Compliance Account - newRegion**] Create Compliance Framework S3 buckets in other region

    ```bash
    aws cloudformation deploy \
        --stack-name compliance \
        --template-file compliance-account-initial-setup.yaml \
        --parameter-overrides $(cat env_var.properties) \
        --no-fail-on-empty-changeset \
        --region <newRegion>
    ```
