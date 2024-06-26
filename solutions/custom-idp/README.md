## AWS Transfer Custom IdP Solution

Content

- [AWS Transfer Custom IdP Solution](#aws-transfer-custom-idp-solution)
- [What is this?](#what-is-this)
- [Features](#features)
- [Architecture](#architecture)
  - [Request flow](#request-flow)
  - [Process flow diagrams](#process-flow-diagrams)
    - [Lambda handler function](#lambda-handler-function)
    - [LDAP module](#ldap-module)
  - [DynamoDB Tables](#dynamodb-tables)
- [Setup Instructions](#setup-instructions)
  - [Prerequisites](#prerequisites)
  - [Deploy the solution](#deploy-the-solution)
  - [Alternative: Automated deployment pipeline](#alternative-automated-deployment-pipeline)
  - [Deploy an AWS Transfer server](#deploy-an-aws-transfer-server)
  - [Define identity providers](#define-identity-providers)
  - [Define Users](#define-users)
  - [(Optional) Define a `$default$` user record](#optional-define-a-default-user-record)
  - [Test the provider](#test-the-provider)
  - [Next steps](#next-steps)
- [Getting Help](#getting-help)
- [Identity provider modules](#identity-provider-modules)
  - [How the identity provider modules work](#how-the-identity-provider-modules-work)
  - [Identity provider module reference](#identity-provider-module-reference)
    - [Argon2](#argon2)
      - [DynamoDB Record Schema](#dynamodb-record-schema)
      - [Parameters](#parameters)
      - [Example](#example)
    - [LDAP and Active Directory](#ldap-and-active-directory)
      - [DynamoDB Record Schema](#dynamodb-record-schema-1)
      - [Parameters](#parameters-1)
      - [Example](#example-1)
    - [Okta](#okta)
      - [DynamoDB Record Schema](#dynamodb-record-schema-2)
      - [Parameters](#parameters-2)
      - [Example](#example-2)
    - [Public Key](#public-key)
      - [DynamoDB Record Schema](#dynamodb-record-schema-3)
      - [Parameters](#parameters-3)
      - [Example](#example-3)
    - [Secrets Manager](#secrets-manager)
- [AWS Transfer session settings inheritance](#aws-transfer-session-settings-inheritance)
  - [user record](#user-record)
- [Modifying/updating solution parameters](#modifyingupdating-solution-parameters)
- [Uninstall the solution](#uninstall-the-solution)
  - [Cleanup remaining artifacts](#cleanup-remaining-artifacts)
- [Logging and troubleshooting](#logging-and-troubleshooting)
  - [Testing custom identity providers](#testing-custom-identity-providers)
  - [Accessing logs](#accessing-logs)
  - [Setting the log level](#setting-the-log-level)
- [FAQ](#faq)
- [Common issues](#common-issues)
- [Tutorials](#tutorials)
  - [Setting up Okta](#setting-up-okta)
  - [Configuring Okta MFA](#configuring-okta-mfa)
  - [Configuring Okta to retrieve session settings from user profile attributes](#configuring-okta-to-retrieve-session-settings-from-user-profile-attributes)
- [Contributing a Module](#contributing-a-module)
- [Security](#security)
- [License](#license)


## What is this?
There are several examples of custom identity providers for AWS Transfer in AWS blog posts an documentation, but there have been no standard patterns for implementing a custom provider that accounts for details including logging and where to store the additional session metadata needed for AWS Transfer, such as the `HomeDirectoryDetails`. This solution provides a reusable foundation for implementing custom identity providers with granular per-user session configuration, and decouples the identity provider authentication logic from the reusable logic that builds a configuration that is returned to AWS Transfer to complete authentication and establish settings for the session. 

## Features
* A standard pattern and DynamoDB schema to store user's IdP and AWS Transfer session settings such as `HomeDirectoryDetails`, `Role`, and `Policy`.
* A standard pattern and DynamoDB schema to store metadata about identity providers and associated settings.
* Support for multiple identity providers connected to a single AWS Transfer Server.
* Support for multiple identity providers for the same username (by using the <IdP>\<username> or <username>@<IdP> conventions at login)
* Connect multiple AWS Transfer servers to the same instantiation of this solution (e.g. to use with both S3 and EFS servers)
* Run multiple instantiations of this solution in the same AWS account.
* Built-in IP allow-list checking.
* Standardized logging patterns with configurable log-level and tracing support.
* Easy-to-deploy infrastructure templates, with step-by-step instructions.
  * The ability to deploy an API Gateway for advanced use cases (e.g. [attaching a WAF WebACL for additional security controls](https://aws.amazon.com/blogs/storage/securing-aws-transfer-family-with-aws-web-application-firewall-and-amazon-api-gateway/))

## Architecture
This section describes the architecture of the solution and introduces the standardized process flow for an authentication request. The diagram below shows the architecture components.

![](diagrams/aws-transfer-custom-idp-solution-high-level-architecture.drawio.png)

### Request flow
1. Client connects to AWS Transfer service, passing credentials. The credentials are then passed to the authentication Lambda function
    
2. The Lambda function performs the following sequence to handle the authentication request:
    1. First, the handler function takes the username (and optionally parses out the identity provider name) and performs a lookup on DynamoDB table. If a matching record exists, it retrieves the record and will use this for the authentication flow. If it does not exist, a '$default$' authentication record is used.
        
    2. Using an `identity_provider_key` field, the Lambda performs a lookup on the `identity_providers` table to obtain IdP information. The `module` field in the response is used to call an idp-specific module. The lambda then passes the parsed username, `identity_provider` and `user` records to the identity provider to continue the authentication flow. 
        
    3. The identity provider module reads the provider-specific settings from the `identity_provider` record it receives. It then uses this to connect to the identity provider (when applicable) and passes the user credentials (when applicable) to authenticate. Since many identity providers are only available on private networks, the Lambda is VPC-attached and uses an ENI in a VPC for private network communication.
        
        **Note: **** **Depending on the logic in the module and the configuration in the `identity_provider` record, the module could retrieve/return additional attributes for making authorization decisions. This is a module-specific implementation detail. For example, the LDAP module supports this.
        
3. After the identity provider module completes authentication, it can make additional authorization decisions based on what is in the user record and its own custom logic. It then finalizes all AWS Transfer session settings (i.e. `Role` and `HomeDirectoryDetails` and returns them to the handler function. The handler function does final validation and returns the response to AWS Transfer.


### Process flow diagrams

#### Lambda handler function

The Lambda handler function itself contains logic for identifying the user and identity provider module to use to perform the authentication. It also checks if the source IP is allowed to initiate an authentication request (based on ipv4_allow_list attribute in each user record) before invoking the target identity provider module.

![](diagrams/aws-transfer-custom-idp-solution-authentication-logic.drawio.png)


#### LDAP module

This is meant to serve as an example of what an individual module would look like. All modules would have the same entrypoint, `handle_auth` and must return a response that is valid to AWS Transfer.

![](diagrams/aws-transfer-custom-idp-solution-ldap-module-process-flow.drawio.png)

### DynamoDB Tables

The solution contains two DynamoDB tables:

* **`${AWS::StackName}_users`**: Contains records for each user, including the associated identity provider to use for authentication and AWS Transfer settings that should be used if authenticated successfully.
* **`${AWS::StackName}_identity_providers`**: Contains details about each identity provider and its associated configuration settings. 

## Setup Instructions
The custom IDP solution can be deployed with one of two methods:

Manually, using the `custom-idp.yaml` Serverless Application Model (SAM) template. If you would like to deploy the solution manually, it is recommended you use the [buildspec_build_deploy.yml](pipeline/buildspec_build_deploy.yml) as a reference for the commands used to build and deploy the solution.

### Prerequisites
* A Virtual Private Cloud (VPC) with private subnets with either internet connectivity via NAT Gateway, or a DynamoDB Gateway Endpoint. 
* Appropriate IAM permissions to deploy the `custom-idp.yaml` CloudFormation template, including but not limited to creating CodePipeline and CodeBuild projects, IAM roles, and IAM policies.

> [!IMPORTANT]  
> The solution must be deployed in the same AWS account and region as the target AWS Transfer servers. 

### Deploy the solution
1. Log into the AWS account you wish to deploy the solution in, switch to the region you will run AWS Transfer in, and start a CloudShell session.

    ![CloudShell session running](screenshots/ss-deploy-01-cloudshell.png)

2. Install the Python 3.11 into your environment.

    ```
    sudo yum install python3.11 python3.11-pip -y
    ```

3. Clone the solution into your environment:
    ```
    cd ~
    git clone https://github.com/aws-samples/toolkit-for-aws-transfer-family.git
    ```
    
4. Run the following command to run the build script, which downloads all package dependencies and generates archives for the Lambda layer and function used in the solution.
    ```
    cd ~/toolkit-for-aws-transfer-family/solutions/custom-idp
    ./build.sh
    ```
    Monitor the execution and verify the script completes successfully.

5. Begin the SAM deployment by using the following command

    ```
    sam deploy --guided --capabilities "CAPABILITY_NAMED_IAM"

    ```
    At the prompts, provide the following information:
    | Parameter | Description | Value |
    | --- | --- | --- |
    | **Stack name** | **REQUIRED**. The name of the CloudFormation stack that will be created. The stack name is also prefixed to several resources that are created to allow the solution to be deployed multiple times in the same AWS account and region. | *your stack name, i.e. transferidp* |
    | **CreateVPC** | **REQUIRED**. Set to *`true`* if you you would like the solution to create a VPC for you, otherwise *`false`* | `true` or `false`|
    | **VPCCIDR** | **CONDITIONALLY REQUIRED**. Must be set if `CreateVPC` is `true`. The CIDR to use for when creating a new VPC. The CIDR should be at least a /24 and will be divided evenly across 4 subnets. Required if CreateVPC is set.  | `true` or `false`|   
    | **VPCId** | **CONDITIONALLY REQUIRED**. Must be set if `CreateVPC` is `false`. The ID of the VPC to deploy the custom IDP solution into. The VPC specified should have network connectivity to any IdPs that will used for authentication.  | *A VPC ID, i.e. `vpc-abc123def456`* |    
    | **Subnets** | **CONDITIONALLY REQUIRED**. Must be set if `CreateVPC` is `false`. A list of subnet IDs to attach the Lambda function to. The Lambda is attached to subnets in order to allow private communication to IdPs such as LDAP servers or Active Directory domain controllers. At least one subnet must be specified, and all subnets must be in the same VPC specified above. **IMPORTANT**: The subnets must be able to reach DynamoDB service endpoints. If using public IdP such as Okta, the subnet must also have a route to a NAT Gateway that can forward requests to the internet. *Using a public subnet will not work because Lambda network interfaces are not assigned public IP addresses*.  | *comma-separated list of subnet IDs, i.e. `subnet-123abc,subnet-456def`* |
    | **SecurityGroups** | **CONDITIONALLY REQUIRED**. Must be set if `CreateVPC` is false. A list of security group IDs to assign to the Lambda function. This is used to control inbound and outbound access to/from the ENI the Lambda function uses for network connectivity. At least one security group must be specified and the security group must belong to the VPC specified above. | *comma-separated list of Security Group IDs, i.e. `sg-abc123`* |
    | **UserNameDelimiter**  | The delimiter to use when specifying both the username and IdP name during login. Supported delimiter formats: <ul><li>[username]**&#64;**[IdP-name]</li><li>[username]**$**[IdP-name]</li><li>[IdP-name]**/**[username]</li><li>[IdP-name]**&#92;&#92;**[username]</li></ul> | One of the following values: <ul><li>**&#64;**</li><li>**$**</li><li>**/**</li><li>**&#92;&#92;**</li></ul> |    
    | **SecretsManagerPermissions** | Set to *`true`* if you will use the Secrets Manager authentication module, otherwise *`false`* | `true` or `false`|
    | **ProvisionApi** | When set to *`true`* an API Gateway REST API will be provisioned and integrated with the custom IdP Lambda. Provisioning and using an API Gateway REST API with AWS Transfer is useful if you intend to [use AWS Web Application Firewall (WAF) WebACLs to restrict requests to authenticate to specific IP addresses](https://aws.amazon.com/blogs/storage/securing-aws-transfer-family-with-aws-web-application-firewall-and-amazon-api-gateway/) or apply rate limiting. | `true` or `false` |    
    | **LogLevel** | Sets the verbosity of logging for the Lambda IdP function. This should be set to `INFO` by default and `DEBUG` when troubleshooting. <br /><br />**IMPORTANT** When set to `DEBUG`, sensitive information may be included in log entries. | `INFO` or `DEBUG` <br /> <br /> *`INFO` recommended as default* |
    | **EnableTracing** | Set to *`true`* if you would like to enable AWS X-Ray tracing on the solution. <br /><br /> Note: X-Ray has additional costs. | `true` or `false` |
    | **UsersTableName** | *Optional*. The name of an *existing* DynamoDB table that contains the details of each AWS Transfer user (i.e. IdP to use, IAM role, logic directory list) are stored. Useful if you have already created a DynamoDB table and records for users **Leave this value empty if you want a table to be created for you**. | *blank* if a new table should be created, otherwise the name of an existing *users* table in DynamoDB |
    | **IdentityProvidersTableName** | *Optional*. The name of an *existing* DynamoDB table that contains the details of each AWS Transfer custom IdPs (i.e. IdP name, server URL, parameters) are stored. Useful if you have already created a DynamoDB table and records for IdPs **Leave this value empty if you want a table to be created for you**. | *blank* if a new table should be created, otherwise the name of an existing *IdPs* table in DynamoDB |  
    | **Confirm changes before deploy** | Prompt to confirm changes after a change set is created. | `y` (default) | 
    | **Allow SAM CLI IAM role creation** | Allow SAM to create a CLI IAM role used for deployments | `y` (default) |
    | **Disable rollback** | Disable rollback if stack creation and resource provisioning fails (can be useful for troubleshooting) | `n` (default) |
    | **Save arguments to configuration file** | Save the parameters specified above to a configuration file for reuse. | `y` (default) | 
    | **SAM configuration file** | The name of the file to save arguments to | `samconfig.toml` (default) |
    | **SAM configuration environment** | The name of the configuration environment to use | `default` (default) |    

### Alternative: Automated deployment pipeline 
An alternate method to deploying the solution is through the `install.yml` CloudFormation template. This template provisions a CodePipeline deployment pipeline linked to this repository. You can also fork this repository to private repo and use that instead. The instructions below walk through this deployment method. 

1. Open and save the [`install.yaml`](install.yaml) template.

2. Log into the AWS account you wish to deploy the solution in, switch to the region you will operate AWS Transfer servers in, and go to the [*Create stack*](https://console.aws.amazon.com/cloudformation/home#/stacks/create) in the CloudFormation console.

3. In the **Create Stack** page, select **Upload a template file**, then click the **Choose file** button and select the `install.yaml` file. Click the **Next** button.

4. On the **Specify stack details** page, complete the parameters using the table below as a guide.

    | Parameter | Description | Value |
    | --- | --- | --- |
    | **Stack name** | **REQUIRED**. The name of the CloudFormation stack that will be created. The stack name is also prefixed to several resources that are created to allow the solution to be deployed multiple times in the same AWS account and region. | *your stack name, i.e. transferidp* |
    | **VPCId** | **CONDITIONALLY REQUIRED**. Must be set if `CreateVPC` is false. The ID of the VPC to deploy the custom IDP solution into. The VPC specified should have network connectivity to any IdPs that will used for authentication.  | *A VPC ID, i.e. `vpc-abc123def456`* |    
    | **Subnets** | **CONDITIONALLY REQUIRED**. A list of subnet IDs to attach the Lambda function to. The Lambda is attached to subnets in order to allow private communication to IdPs such as LDAP servers or Active Directory domain controllers. At least one subnet must be specified, and all subnets must be in the same VPC specified above. **IMPORTANT**: The subnets must be able to reach DynamoDB service endpoints. If using public IdP such as Okta, the subnet must also have a route to a NAT Gateway that can forward requests to the internet. *Using a public subnet will not work because Lambda network interfaces are not assigned public IP addresses*.  | *comma-separated list of subnet IDs, i.e. `subnet-123abc,subnet-456def`* |
    | **SecurityGroups** | **CONDITIONALLY REQUIRED**. Must be set if `CreateVPC` is false. A list of security group IDs to assign to the Lambda function. This is used to control inbound and outbound access to/from the ENI the Lambda function uses for network connectivity. At least one security group must be specified and the security group must belong to the VPC specified above. | *comma-separated list of Security Group IDs, i.e. `sg-abc123`* |
    | **UserNameDelimiter**  | The delimiter to use when specifying both the username and IdP name during login. Supported delimiter formats: <ul><li>[username]**&#64;**[IdP-name]</li><li>[username]**$**[IdP-name]</li><li>[IdP-name]**/**[username]</li><li>[IdP-name]**&#92;&#92;**[username]</li></ul> | One of the following values: <ul><li>**&#64;**</li><li>**$**</li><li>**/**</li><li>**&#92;&#92;**</li></ul> |    
    | **SecretsManagerPermissions** | Set to *`true`* if you will use the Secrets Manager authentication module, otherwise *`false`* | `true` or `false`|
    | **ProvisionApi** | When set to *`true`* an API Gateway REST API will be provisioned and integrated with the custom IdP Lambda. Provisioning and using an API Gateway REST API with AWS Transfer is useful if you intend to [use AWS Web Application Firewall (WAF) WebACLs to restrict requests to authenticate to specific IP addresses](https://aws.amazon.com/blogs/storage/securing-aws-transfer-family-with-aws-web-application-firewall-and-amazon-api-gateway/) or apply rate limiting. | `true` or `false` |    
    | **LogLevel** | Sets the verbosity of logging for the Lambda IdP function. This should be set to `INFO` by default and `DEBUG` when troubleshooting. <br /><br />**IMPORTANT** When set to `DEBUG`, sensitive information may be included in log entries. | `INFO` or `DEBUG` <br /> <br /> *`INFO` recommended as default* |
    | **EnableTracing** | Set to *`true`* if you would like to enable AWS X-Ray tracing on the solution. <br /><br /> Note: X-Ray has additional costs. | `true` or `false` |
    | **UsersTableName** | *Optional*. The name of an *existing* DynamoDB table that contains the details of each AWS Transfer user (i.e. IdP to use, IAM role, logic directory list) are stored. Useful if you have already created a DynamoDB table and records for users **Leave this value empty if you want a table to be created for you**. | *blank* if a new table should be created, otherwise the name of an existing *users* table in DynamoDB |
    | **IdentityProvidersTableName** | *Optional*. The name of an *existing* DynamoDB table that contains the details of each AWS Transfer custom IdPs (i.e. IdP name, server URL, parameters) are stored. Useful if you have already created a DynamoDB table and records for IdPs **Leave this value empty if you want a table to be created for you**. | *blank* if a new table should be created, otherwise the name of an existing *IdPs* table in DynamoDB |
    | **Repo** | *Optional*. The name of repository to retrieve the custom IdP solution code from during deployment. Default is to use the public repo on Github. **Only change this if you have forked the repo for customization** | `toolkit-for-aws-transfer-family` if using the public aws-samples repo, otherwise the name of your repo. |
    | **RepoOwner** | *Optional*. The owner of the repository where the custom IdP solution code is retrieved from during deployment. Default is `aws-samples` on Github. **Only change this if you have forked the repo for customization** | `aws-samples` if using the public aws-samples repo, otherwise the name of your repo. |  
    | **CodeStarConnectionArn** | *Optional*. The ARN of the [CodeStar/Developer Tools Connection](https://docs.aws.amazon.com/dtconsole/latest/userguide/connections.html) that will be used to access the repo. Default is blank, because a connection is not needed to access a public repo on Github. **Only change this if you have forked the repo for customization and are using a private repo.** | *blank* if using the public aws-samples repo, otherwise the ARN of the Connection, i.e. `arn:aws:codestar-connections:[REGION]:[ACCOUNTID]:connection/8d27eabf-90c9-4d52-b211-e504427e101b`. | 
    | **ProjectSubfolder** | *Optional*. The path to the custom IdP solution source code within the repo. In the *toolkit-for-aws-transfer-family* repo, this should always be `solutions/custom-idp`. **Only change this if you have forked the repo and moved the source code.** | `solutions/custom-idp` if using the public aws-samples repo. |     
    
    Once the parameters are set appropriately, click the **Next** button.

5.  At the **Configure stack options** screen, scroll to the bottom and click the **Next** button.

6.  At the **Review and create** screen, review all settings then scroll to the bottom, click the box next to *I acknowledge that AWS CloudFormation might create IAM resources with custom names*, then click **Next**.

7.  Once the stack creation completes, go to **Outputs** tab in the console and click the link next to the **Pipeline** key to go to the CodePipeline. This pipeline will deploy retrieve the source code, then build and deploy the SAM template using a CodeBuild project. This will then deploy an additional CloudFormation stack that provisions the custom IdP solution components. 

    Wait for the pipeline to complete successfully. If an error occurs, you can use the **View details** buttons to review logs and error messages and troubleshoot.


8.  After the pipeline completes, go back to the CloudFormation Console, select the *[StackName]-awstransfer-custom-idp* stack, and click the **Outputs** button, copy/paste the CloudFormation outputs to a text editor and save them for future use. Below are descriptions of the outputs:

    | Key | Description | Value |
    | --- | --- | --- |
    | `IdpHandlerFunction` | The ARN of the Lambda function. Use this ARN when configuring an AWS Transfer server to use a Lambda-based custom identity provider | `arn:${AWS::Partition}:lambda:${AWS::Region}:${AWS::AccountId}:function:${AWS::StackName}_awstransfer_idp` |
    | `IdpHandlerLogGroupUrl` | A URL that will take you to the IdpHandler function's Cloudwatch Log group. These logs can be useful for troubleshooting IdP errors. | `https://${AWS::Region}.console.aws.amazon.com/cloudwatch/home?region=${AWS::Region}#logsV2:log-groups/log-group/$252Faws$252Flambda$252F${IdpHandlerFunction}` |
    | `ApiUrl` | *Optional* A URL to the API Gateway REST API that was provisioned. Use this URL when configuring an AWS Transfer server to use an REST API-based custom identity provider. This output is only displayed when `ProvisionApi` is set to `true` | `https://${CustomIdentityProviderApi}.execute-api.${AWS::Region}.amazonaws.com/${ApiStage}` |
    | `ApiRole` | *Optional* The name of an IAM role created for AWS Transfer to use when invoking the REST API. This is used when configuring an AWS Transfer server to use an REST API-based custom identity provider. This output is only displayed when `ProvisionApi` is set to `true` | `${AWS::StackName}_TransferApiRole` |
    

### Deploy an AWS Transfer server

***Note***: If you have an existing AWS Transfer server configured to use a custom identity provider, it can be modified to use the Lambda function or API Gateway REST API instead of creating a new server by going to the AWS Transfer console, selecting the server, and clicking **Edit** next to the *Identity provider** section. If the server was not configured with a custom identity provider, or if you wish to switch between Lambda and API Gateway based providers, you will need to re-provision your AWS Transfer server.

1. Go to the AWS Transfer console in the region where the solution is deployed and click **Create server**.
   
  ![AWS Transfer console](screenshots/ss-transfer-01-create-server.png)

2. At the **Choose protocols** screen, select the protocols to enable and click **Next**

3. At the **Choose an identity provider** screen, select **Custom Identity Provider**, then select one of the identity provider options below.

    **Option 1: Use AWS Lambda to connect your identity provider**
    
    Choose the AWS Lambda function from the list that matches the output from **`IdpHandlerFunction`** (i.e. `[StackName]_awstransfer_idp`)
    
    ![AWS Transfer console](screenshots/ss-transfer-03-choose-idp-lambda.png)

    **Option 2: Use API Gateway to connect your identity provider**
    
    * Specify the API Gateway Url from the `ApiUrl` output (i.e. `https://[API].execute-api.[REGION].amazonaws.com/prod`)
    * Choose the IAM role from the list that matches the output from the **`ApiRole`** (i.e. `[StackName]_TransferApiRole`)

        ![AWS Transfer console](screenshots/ss-transfer-03-choose-idp-api.png)    

4. At the **Choose an endpoint** screen, configure the AWS transfer endpoint type and hostname, then click **Next**.
   
5. At the **Choose a domain** screen, select the AWS Storage Service to use (S3 or EFS) and click **Next**.

6. At the **Configure additional details** screen, configure any additional settings for the AWS Transfer server and click **Next**.

7. On the final screen, review and verify all settings for the new server and click **Create**.

8. At the AWS Transfer console, a new server will appear in the list and show status **Starting**. Wait for the process to complete and then proceed to configure Identity Providers.
   
    ![AWS Transfer console](screenshots/ss-transfer-07-starting.png)

### Define identity providers
To get started, you must define one or more identity providers in the DynamoDB table `$[StackName]_identity_providers`. Each identity provider record stores the configuration and identity provider module to use for authenticating users mapped to that provider. For complete details on identity provider modules, settings, and examples see the [Identity Provider Modules](#identity-provider-modules). To get started, this section will define an identity provider that uses the `public_key` module. 

> [!NOTE]  
> The `public_key` module is supported with AWS Transfer Servers that are configured with SFTP protocol only. If your server us using a different protocol, you should configure a different provider.

1. In the AWS Console, navigate to the DynamoDB console, select **Tables > Explore** items on the sidebar. Select the `[StackName]_identity_providers` table, then click **Create Item**.

  ![DynamoDB identity providers table](screenshots/ss-identityprovidersetup-01-dynamodb-table.png)

2. In the **Create Item** screen, click **JSON View**, then paste the following into the record:
  ```json
  {
    "provider": {
      "S": "publickeys"
    },
    "public_key_support": {
      "BOOL": true
    },        
    "config": {
      "M": {
      }
    },
    "module": {
      "S": "public_key"
    }
  }
  ```

1. Click the **Form** button to switch back to Form view. The attributes of the record will be displayed as shown in the screenshot below. 

   ![DynamoDB identity provider record for publickeys](screenshots/ss-identityprovidersetup-02-publickeys-record.png)

     * The `provider` attribute contains the name defined of the identity provider. In this case, it is `publickeys` but it could be more meaningful such as `devteam-publickeys` or `example.com` if it were an external identity provider. 
     * The `module` attribute contains the name of the identity provider module to use, in this case `public_key`. 
     * The `config` is an attribute map that is used for storing identity provider configuration details. This `public_key` module is very simple and therefore has no configuration settings. Other modules such as LDAP and Okta store settings in this attribute map.

2. After reviewing, click **Create Item**. The first identity provider has now been defined. Next, we'll begin defining users.

### Define Users

Once identity providers are defined, user records must be created. Each user records may contain the settings that will be used for an AWS Transfer session and can also contain public keys when using the `public_key` module or for AWS Transfer servers configured with Public Key AND Password support. Each record also maps the username to a given identity provider. In this section, we will create a user record and map it to the `publickeys` identity provider created in the previous section.

> [!IMPORTANT]  
> All usernames specified in the `[StackName]_users` must be entered as lowercase.
>

1. As a prerequisite, a public/private key pair must be generated for use with the `public_key` module. If you do not already have a key pair, you can follow the [Generate SSH Keys procedure](https://docs.aws.amazon.com/transfer/latest/userguide/key-management.html#sshkeygen) from the AWS Transfer documentation to generate them. You do NOT need to create a service managed user and enter the public key - please skip those steps.

2. Navigate to the [DynamoDB console](https://console.aws.amazon.com/dynamodb), select **Tables > Explore** items from the sidebar. Select the `[StackName]_users` table, then click **Create Item**.

  ![DynamoDB users table](screenshots/ss-usersetup-01-dynamodb-table.png)

3. In the **Create Item** screen, click **JSON View**, then paste the following into the record:
  ```json
  {
    "user": {
      "S": "johnsmith"
    },
    "identity_provider_key": {
      "S": "publickeys"
    },  
    "config": {
      "M": {
        "HomeDirectoryDetails": {
          "L": [
            {
              "M": {
                "Entry": {
                  "S": "/s3files"
                },
                "Target": {
                  "S": "/[bucketname]/prefix/to/files"
                }
              }
            },
            {
              "M": {
                "Entry": {
                  "S": "/efs"
                },
                "Target": {
                  "S": "/fs-[efs-fs-id]"
                }
              }
            }
          ]
        },
        "HomeDirectoryType": {
          "S": "LOGICAL"
        },
        "PosixProfile": {
          "M": {
            "Gid": {
              "S": "1000"
            },
            "Uid": {
              "S": "1000"
            }
          }
        },
        "PublicKeys": {
          "SS": [
            "ssh-ed25519 [PUBLICKEY]",
            "ssh-rsa [PUBLICKEY]"
          ]
        },
        "Role": {
          "S": "arn:aws:iam::[AWS Account Id]:role/[Role Name]"
        }
      }
    },
    "ipv4_allow_list": {
      "SS": [
        "0.0.0.0/0"
      ]
    }
  }
  ```


4. Click the **Form** button to switch back to Form view and expand all of the nested attributes. The attributes of the record will be displayed as shown in the screenshot below. Below are the details of the various fields:

     * The `user` key contains the username that will be passed to AWS Transfer during authentication. This will be used to lookup the user record. **In order for lookups to success, this value must ALWAYS be lowercase.**
     * The `identity_provider_key` attribute contains the identity provider name from the `[StackName]_identity_providers` table. In this case, it is the `publickeys` provider created in the previous section. Note that this is the name of the identity provider, *not* the name of the identity provider module. 
     * The `ipv4_allow_list` attribute is a list of remote IP CIDRs that are allowed to authenticate as the user. This is an optional attribute and by default all remote IPs are allowed to authenticate as the user. 
     * The `config` attribute is a mapping of the user's session settings. Its values follow the same format as those found in the [Lambda Values Section](https://docs.aws.amazon.com/transfer/latest/userguide/custom-identity-provider-users.html#event-message-structure) of the custom identity provider documentation). This includes the `HomeDirectoryType`, `HomeDirectoryDetails` (for logical directory mappings),`PosixProfile`, and any `PublicKeys` associated with the user. Note that `PublicKeys` is an optional field, depending on the identity provider and AWS Transfer authentication method.
       * Note that `HomeDirectoryDetails` can have both S3 and EFS targets, in the scenario you wish to use the solution with both an S3 and EFS AWS Transfer server.

    ![DynamoDB users table](screenshots/ss-usersetup-02-user-record.png)

5. Modify the user configuration details to reflect your environment. Specifically, you should set:
  * `HomeDirectoryDetails`: Modify this list, setting the `Target` values to S3 buckets or EFS filesystems.
  * `PosixProfile`: If connecting to an AWS Transfer server attached to EFS, change the `Uid` and `Gid` to reflect those belonging to the user. Note that this is not required for Transfer servers attached to S3.
  * `PublicKeys`: Since this user will be authenticated with the `public_key` module, provide one or more valid public keys that will be used to authenticate the user (e.g. the contents of the `.pub` key generate in step 1 of this section). Each public key should be its own entry in the `PublicKeys`.
  * `Role`: Specify the AWS Transfer IAM Role that will be used to access data in S3 and EFS. Remember: This role must have a trust policy that gives the AWS Transfer service permission to assume it and must have the correct policies attached for accessing the data in S3 or EFS. For more information, refer to the [AWS Transfer documentation](https://docs.aws.amazon.com/transfer/latest/userguide/requirements-roles.html).


6. After reviewing, click **Create Item**. The first user, `joesmith`, has been created and mapped to the `publickeys`. Next we can (optionally) create a *default* user and finally test authentication.

### (Optional) Define a `$default$` user record
While this solution is designed to provide very granular and flexible authentication and authorization to AWS Transfer users to support a variety of use cases, there are use cases where all users will access the same identity provider and either apply the same AWS Transfer session configuration such as `Role`, `PosixProfile`, and `Policy`, or retrieve session configuration parameters dynamically from the source identity provider itself. The `$default$` user record is designed to support these scenarios. The `$default$` user record is used when the Lambda function is unable to find a user record that matches the username received in the request. 

Here are some important considerations for this record:
* If no `$default$` record is specified, authentication will simply fail (which may be intended).  
* Unless the identity provider overrides them, all users that authenticate with the `$default$` user will receive the session settings (i.e. `Role`, `PosixProfile`, and `Policy`) defined in the record. If more granularity is desired, you should define individual user records for each record.
* Be careful when specifying a `$default$` record when there are other user records specified. Keep in mind that any username that doesn't match will attempt to use `$default` which could produce unexpected session access.

Below is an example of a `$default$` user record, for mapped to an an Active Directory or LDAP identity provider. 

```json
{
  "user": {
    "S": "$default$"
  },
  "identity_provider_key": {
    "S": "example.com"
  },
  "config": {
    "M": {
      "HomeDirectoryDetails": {
        "L": [
          {
            "M": {
              "Entry": {
                "S": "/pics"
              },
              "Target": {
                "S": "/[bucket name]/pictures"
              }
            }
          },
          {
            "M": {
              "Entry": {
                "S": "/files"
              },
              "Target": {
                "S": "/[bucket name]/files"
              }
            }
          }
        ]
      },
      "HomeDirectoryType": {
        "S": "LOGICAL"
      },
      "Role": {
        "S": "arn:aws:iam::[aws account id]:role/[role name]"
      }
    }
  },
  "ipv4_allow_list": {
    "SS": [
      "0.0.0.0/0"
    ]
  }
}
```
To further illustrate a scenario where `$default$` is used, suppose `PosixProfile` and a scoped `Policy` are retrieved from Active Directory or LDAP server. This is what the identity provider record might look like.

```json
{
  "provider": {
    "S": "example.com"
  },
  "config": {
    "M": {
      "attributes": {
        "M": {
          "Gid": {
            "S": "gidNumber"
          },
          "Policy": {
            "S": "comment"
          },
          "Uid": {
            "S": "uidNumber"
          }
        }
      },
      "port": {
        "N": "636"
      },
      "search_base": {
        "S": "DC=EXAMPLE,DC=COM"
      },
      "server": {
        "S": "dc1.example.com"
      },
      "ssl": {
        "BOOL": true
      },
      "ssl_verify": {
        "BOOL": false
      }
    }
  },
  "module": {
    "S": "ldap"
  }
}
```
The record above dynamically maps Active Directory/LDAP attributes `uidNumber`, `gidNumber`, and `comments` to `Uid`, `Gid`, and scopedown `Policy` in the AWS Transfer session configuration.


### Test the provider
To test the identity provider `publickeys` and user `joesmith` created in the previous sections, use an SFTP client to connect to the AWS Transfer server. For example, on a Linux or Mac client with `sftp` client installed open a terminal window and enter the command to connect:
```bash
  sftp -i path/to/privatekey johnsmith@[transfer-server-endpoint-address]
```

Below is an example of a successful connection and file/directory listing:

![SFTP terminal](screenshots/ss-testprovider-01-terminal.jpg)

To view the provider logs, open Cloudwatch Logs and select the log group `/aws/lambda/[StackName]_awstransfer_idp`. The log streams will show the login event and details about the user and identity provider evaluation. If there was a failure, you will also see error messages and exceptions in this log. Below is a screenshot showing the logging in near realtime using [Cloudwatch Live Tail](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogs_LiveTail.html). 

![Cloudwatch Live Tail](screenshots/ss-testprovider-02-logs.png)

> [!NOTE]  
> If the Lambda logs don't show failures but SSH key authentication still fails and/or prompts for a password, it's possible that AWS Transfer did not successfully verify the supplied private key against the public key. It's also possible that required session properties were missing or misconfigured in the user record. Check the [AWS Transfer server log group](https://docs.aws.amazon.com/transfer/latest/userguide/structured-logging.html) for additional details.

### Next steps
With the solution setup completed and tested, you can begin adding more identity provider and user records, and explore advanced functionality in each module to support your use case. The [identity provider modules](#identity-provider-modules) section provides detailed information about each identity provider, its configuration settings, and example configurations. 


## Getting Help

The best way to interact with our team is through GitHub. You can open an [issue](https://github.com/aws/tookit-for-aws-transfer-family/issues/new/choose) and choose from one of our templates for bug reports, feature request, etc.

You may also find help on [AWS re:Post](https://repost.aws). When asking a question, tag it with `AWS Transfer Family`.
 
## Identity provider modules
This section describes how the identity provider modules work and describes the module-specific parameters that are used in each module's configuration. 

### How the identity provider modules work
In the `users` table, each user has a corresponding `provider` property that indicates the provider from in the `identity_providers` table that should be used for authentication. 
1. When a user initiates authentication, the user record is retrieved and the `provider` value is used to lookup corresponding record in the `identity_providers` table. 
2. The `module` value is used to load the corresponding identity provider module (stored in the `idp_handler/idp_modules` directory of the source code)
3. The `handle_auth` function is called, passing the configurations stored in both the user and identity provider records to the module for it to use.

Any module-specific settings are stored in the `config` value of the record within the `identity_providers` table. This is a DynamoDB `Map` value that can contain multiple nested values. 


### Identity provider module reference

#### Argon2
The Argon2 module allows you to generate and use [Argon2](https://en.wikipedia.org/wiki/Argon2) hashed passwords that are stored in the `user` record for authentication. 

This module serves as a method to define "local" user credentials within the custom idp solution. It is not recommended that this method be used at any scale in a production environment since this solution does not contain self-service password management functionality for end users.

> [!NOTE]  
> To use this module, you must generate an argon2 hash and store it in an `argon2_hash` field of the `config` for each `user` record. See the **Examples** section below for more details.

##### DynamoDB Record Schema
```json
{
  "provider": {
    "S": "[provider name]"
  },
  "config": {
  },
  "module": {
    "S": "argon2"
  }     
}
```
##### Parameters

**provider**

A name used for referencing the provider in the `users` table. This value is also used when users specify an identity provider during authentication (e.g. `username@provider`).

**Type:** String

**Constraints:** None

**Required:** Yes

**module**

The name of the module that will be loaded to perform authentication. **This should be set to `argon2`.**

Type: String

Constraints: None

Required: Yes

##### Example

The following example configures the argon2 provider with a provider name `local_password`. 

```json
{
  "provider": {
    "S": "local_password"
  },
  "config": {
    "M": {
    }
  },
  "module": {
    "S": "argon2"
  }
}
```

The following is an example of a `user` record that uses the `argon2` identity provider defined above. This shows the `argon2_hash` field that stores the password hash.

```json
{
    "user": {
        "S": "johnsmith"
    },
    "identity_provider_key": {
        "S": "local_password"
    },
    "config": {
      "argon2_hash": {
        "S": "$argon2i$v=19$m=4096,t=3,p=$argon2i$v=19$m=4096,t=3,p=1$Q1JYWUZvSExndGwxVFBKVDdnUUlUMXpCVlpjTUJibbbbbbbb+2/GwZZmGUN3UiclEIXWX3bbbbbbbbb"
      },
        "M": {
            "HomeDirectoryDetails": {
                "L": [{
                        "M": {
                            "Entry": {
                                "S": "/home"
                            },
                            "Target": {
                                "S": "organization-bucket/users/johnsmith"
                            }
                        }
                    },
                    {
                        "M": {
                            "Entry": {
                                "S": "/finance"
                            },
                            "Target": {
                                "S": "organization-bucket/departments/finance"
                            }
                        }
                    }
                ]
            },
            "HomeDirectoryType": {
                "S": "LOGICAL"
            }
        }
    },
    "ipv4_allow_list": {
        "SS": [
            "172.31.0.0/16",
            "192.168.10.0/24"
        ]
    }
}
```
tr -dc 'A-Za-z0-9!"#$%&'\''()*+,-./:;<=>?@[\]^_`{|}~' </dev/urandom | head -c 32; echo
On system with the `argon2` package/binaries installed, a hash can be generated for testing using this command: 
```bash
unset -v password; set +o allexport; echo "Enter password"; IFS= read -rs password < /dev/tty; printf '%s' "$password" | argon2 $(head /dev/urandom | LC_ALL=C tr -dc A-Za-z0-9 | head -c 32; echo) -e; unset -v password;
```
Copy the hash and paste it into the `argon_hash` field in the `user` record above. 

#### LDAP and Active Directory
The `ldap` module supports authentication with Active Directory and LDAP servers. Both LDAP and LDAPS are supported. User attributes can be retrieved and mapped to the server response, such as `Uid` and `Gid`. 

##### DynamoDB Record Schema
```json
{
  "provider": {
    "S": "[provider name]"
  },
  "config": {
    "M": {
      "attributes": {
        "M": {
          "Gid": {
            "S": "[LDAP attribute name]"
          },
          "Uid": {
            "S": "[LDAP attribute name]"
          },          
          "Role": {
            "S": "[LDAP attribute name]"
          },
          "Policy": {
            "S": "[LDAP attribute name]"
          }
        }
      },
      "ignore_missing_attributes": {
        "BOOL": [true or false]
      },
      "port": {
        "N": "[port number]"
      },
      "search_base": {
        "S": "LDAP search base"
      },
      "server": {
        "S": "[LDAP server address]"
      },
      "ssl": {
        "BOOL": [true or false]
      },
      "ssl_verify": {
        "BOOL": [true or false]
      }
    }
  },
  "module": {
    "S": "ldap"
  }
}
```
##### Parameters

**provider**

A name used for referencing the provider in the `users` table. This value is also used when users specify an identity provider during authentication (e.g. `username@provider`).

**Type:** String

**Constraints:** None

**Required:** Yes

**module**

The name of the LDAP module that will be loaded to perform authentication. **This should be set to `ldap`.**

Type: String

Constraints: None

Required: Yes

**config/server**

The DNS address or IP address of the LDAP server or domain to connect to for authentication.

Type: String

Constraints: Must be a FQDN or IP address

Required: Yes

**config/search_base**

The LDAP search base to when connecting to LDAP and to lookup any users/attributes in. For example, in AD domain `EXAMPLE.COM` it may be `DC=EXAMPLE,DC=COM`. 

Type: String

Constraints: Must be a valid LDAP search base.

Required: Yes

**config/port**

The port number of the LDAP or Active Directory server to connect to. This is typically `389` for non-SSL and `636` for SSL.

Type: Number

Constraints: Must be a valid port number

Required: No

Default: `636`

**config/ssl**

Determines if an SSL connection should be established to the server. SSL is enabled by default.

Type: Boolean

Constraints: Must be `true` or `false`

Required: No

Default: `true`

**config/ssl_verify**

When set to `true` and connecting with SSL, the identity of the server will be validated against the address used in the `config/server` value and that the certificate is valid. Set to `false` if the server name does not match the DNS address used or has a self-signed certificate (i.e. for testing). 

Type: Boolean

Constraints: Must be `true` or `false`

Required: No

Default: `true`

**config/attributes**

An optional key/value map of AWS Transfer user attributes and the corresponding LDAP or AD attributes that should be retrieved and used for them. 

For example, if you wish to pass a `Uid` and `Gid` from AD or LDAP to AWS Transfer to use in `PosixProfile` and the values are stored in `UidNumber` and `GidNumber` attributes in LDAP or AD, the entry would be:
```json
"attributes": {
  "M": {
      "Gid": {
      "S": "gidNumber"
      },
      "Uid": {
      "S": "uidNumber"
      }
  }
}
```
> [!NOTE]  
> Any attributes returned will override corresponding values that have been specified in the user's record from the `users` table. 

Type: Map

Constraints: Only attribute keys `Gid`, `Uid`, `Policy`, and `Role` are supported.

Required: No

Default: *none*

**config/ignore_missing_attributes**

When set to `true`, any LDAP or AD attributes that return no value in the `attributes` map will be ignored. Otherwise, the authentication is considered a failure.

When enabled the value is missing, any corresponding values that have been specified in the user's record from the `users` table will be used. 

> [!NOTE]  
> It is recommended this be set to `false`, since missing or empty attributes could indicate the user's LDAP or AD profile has not been correctly configured and an empty attribute such as `Policy` could provide less restrictive access than desired. 

Type: Boolean

Constraints: Must be `true` or `false`

Required: No

Default: `false`

##### Example
```json
{
  "provider": {
    "S": "example.com"
  },
  "config": {
    "M": {
      "attributes": {
        "M": {
          "Gid": {
            "S": "gidNumber"
          },
          "Role": {
            "S": "comment"
          },
          "Uid": {
            "S": "uidNumber"
          }
        }
      },
      "ignore_missing_attributes": {
        "BOOL": false
      },
      "port": {
        "N": "636"
      },
      "search_base": {
        "S": "DC=example,DC=com"
      },
      "server": {
        "S": "ldap.example.com"
      },
      "ssl": {
        "BOOL": true
      },
      "ssl_verify": {
        "BOOL": true
      }
     }
  },
  "module": {
    "S": "ldap"
  }
}
```
#### Okta

The `okta` module supports authentication with an Okta instance. It supports TOTP-based MFA and can optionally retrieve user profile attributes and map them to session settings such as `Uid` and `Gid`. 

##### DynamoDB Record Schema
```json
{
  "provider": {
    "S": "[provider name]"
  },
  "config": {
    "M": {
      "attributes": {
        "M": {
          "Gid": {
            "S": "[Okta profile attribute name]"
          },
          "Uid": {
            "S": "[Okta attribute name]"
          },          
          "Role": {
            "S": "[Okta attribute name]"
          },
          "Policy": {
            "S": "[Okta attribute name]"
          }
        }
      },
      "ignore_missing_attributes": {
        "BOOL": [true or false]
      },
      "mfa_token_length": {
        "N": "[token length]"
      },
      "okta_domain": {
        "S": "[FQDN Okta domain]"
      },
      "okta_app_client_id": {
        "S": "[app client id]"
      },
      "okta_redirect_uri": {
        "S": "[okta redirect uri]"
      },      
      "mfa": {
        "BOOL": [true or false]
      }
    }
  },
  "module": {
    "S": "okta"
  }
}
```
##### Parameters

**provider**

A name used for referencing the provider in the `users` table. This value is also used when users specify an identity provider during authentication (e.g. `username@provider`).

**Type:** String

**Constraints:** None

**Required:** Yes

**module**

The name of the Okta module that will be loaded to perform authentication. **This should be set to `okta`.**

Type: String

Constraints: None

Required: Yes

**config/okta_domain**

The DNS address or IP address of the Okta domain to connect to for authentication.

Type: String

Constraints: Must be a FQDN or IP address

Required: Yes

**config/okta_app_client_id**

The Client ID of the Okta application that will be used to obtain a session cookie and retrieve user profile attributes. **Only required if Okta user profile attributes will be retrieved from Okta. The Okta application must be configured with Okta API scope `okta.users.read.self`. 

Type: String

Constraints: Must be a valid Client ID associated with a native Okta application. 

Required: No.

**config/okta_redirect_uri**

A "Sign-in redirect URI" that will be passed in the request to retrieve a session cookie for the Okta application. Each Okta application defines a valid list of redirect URIs that clients are allowed to be redirected to after authentication. The URI does not have to be a valid website, but the URI passed in the request must match the list of URIs allowed by the application in order for a session cookie to be returned.

Type: Boolean

Constraints: Must be a valid "Sign-in redirect URI" that is allowed in login requests for the Okta application. 

Required: No

Default: `awstransfer:/callback`

**config/mfa**

When set to `true`, indicates Okta is configured to required MFA. When enabled, users must enter their password plus the temporary one time code when prompted for their password (e.g. `password123456`)

Only TOTP MFA is supported by this module.

Type: Boolean

Constraints: Must be `true` or `false`

Required: No

Default: `false`

**config/mfa_token_length**

The number of digits to expect in the MFA token that is appended to the password. By default, a 6-digit code is assumed.

Type: Integer

Constraints: Must be an integer greater than zero

Required: No

Default: `6`

**config/attributes**

An optional key/value map of AWS Transfer user attributes and the corresponding Okta user profile attributes that should be retrieved and used for them. When this value is set, `okta_app_client_id` must be set to retrieve user profile attributes from Okta.

For example, if you wish to pass a `Uid` and `Gid` from Okta to AWS Transfer to use in `PosixProfile` and the values are stored in `UidNumber` and `GidNumber` attributes in the Okta user's profile, the entry would be:
```json
"attributes": {
  "M": {
      "Gid": {
      "S": "gidNumber"
      },
      "Uid": {
      "S": "uidNumber"
      }
  }
}
```
> [!NOTE]  
> Any attributes returned will override corresponding values that have been specified in the user's record from the `users` table. 

Type: Map

Constraints: Only attribute keys `Gid`, `Uid`, `Policy`, and `Role` are supported.

Required: No

Default: *none*

**config/ignore_missing_attributes**

When set to `true`, any Okta user profile attributes that return no value in the `attributes` map will be ignored. Otherwise, the authentication is considered a failure.

When enabled the value is missing, any corresponding values that have been specified in the user's record from the `users` table will be used. 

> [!NOTE]  
> It is recommended this be set to `false`, since missing or empty attributes could indicate the user's Okta profile has not been correctly configured and an empty attribute such as `Policy` could provide less restrictive access than desired. 

Type: Boolean

Constraints: Must be `true` or `false`

Required: No

Default: `false`

##### Example

The following example identity provider record configures the Okta module to:
* Connect to the okta domain `dev-xxxxx.okta.com`
* Enable MFA with a 6-digit token
* Retrieve Gid, Uid, Role, and Policy attributes from Okta user profile attributes


```json
{
  "provider": {
    "S": "okta.example.com"
  },
  "config": {
    "M": {
      "attributes": {
        "M": {
          "Gid": {
            "S": "gidNumber"
          },
          "Uid": {
            "S": "uidNumber"
          },          
          "Role": {
            "S": "AWSTransferRole"
          },
          "Policy": {
            "S": "AWSTransferScopeDownPolicy"
          }
        }
      },
      "ignore_missing_attributes": {
        "BOOL": false
      },
      "mfa_token_length": {
        "N": "6"
      },
      "okta_domain": {
        "S": "dev-xxxxx.okta.com"
      },
      "okta_app_client_id": {
        "S": "0123abcDE456f78gH9"
      },
      "okta_redirect_uri": {
        "S": "callback:/awstransfer"
      },      
      "mfa": {
        "BOOL": true
      }
    }
  },
  "module": {
    "S": "okta"
  }
}
```

#### Public Key
The Public Key module is is used to perform authentication with public/private key pairs. The module itself *does not* perform this validation - it simply verifies that `PublicKeys` for the user are included in the response to AWS Transfer so that it can complete validation of the private key. There are no settings to configure.

##### DynamoDB Record Schema
```json
{
  "provider": {
    "S": "[provider name]"
  },
  "config": {
  },
  "module": {
    "S": "public_key"
  },
  "public_key_support": {
    "BOOL": true
  }     
}
```
##### Parameters

**provider**

A name used for referencing the provider in the `users` table. This value is also used when users specify an identity provider during authentication (e.g. `username@provider`).

**Type:** String

**Constraints:** None

**Required:** Yes

**module**

The name of the public key module that will be loaded to perform authentication. **This should be set to `public_key`.**

Type: String

Constraints: None

Required: Yes

**module**

The name of the public key module that will be loaded to perform authentication. **This should be set to `public_key`.**

Type: String

Constraints: None

Required: Yes

**public_key_support**

Indicates that the identity provider supports handling public key authentication. **This should always be set to `true` for this module.**

**Type:** Boolean

**Constraints:** None

**Required:** Yes

##### Example

The following example configures the public key provider with a provider name `publickeys`. 

```json
{
  "provider": {
    "S": "publickeys"
  },
  "config": {
    "M": {
    }
  },
  "module": {
    "S": "public_key"
  }
}
```
#### Secrets Manager

TODO 

## AWS Transfer session settings inheritance
When an AWS Transfer Family custom identity provider authenticates a user, it returns all session setup properties such as the `HomeDirectoryDetails`, `Role`, and `PosixProfile` . To maximize the flexibility of this solution, most of those values can can be specified in the user record, identity provider record, as well as from the identity provider itself (i.e. LDAP attributes). When a value is contained is multiple sources, there is an ordered inheritance/priority to merge the final values together, with 1 being the highest priority:

1. Values returned by the identity provider. 

    **Note:** Each identity provider module is responsible for the logic to apply/override values.
2. Values in `config` field of the user record `users` table
3. values in the `config` of the identity provider record in the `identity_providers` table

**Example Scenario:** An organization wishes to setup an LDAP identity provider. They want the AWS Transfer server to use `UidNumber` and `GidNumber` attributes from the LDAP server, have all users for that identity provider share the same `Role`, and specify all other settings on a per-user basis. This is what the corresponding `user` and `identity_provider` records might look like:

**identity_provider record**
```json
{
    "provider": {
        "S": "example.com"
    },
    "config": {
        "M": {
            "attributes": {
                "M": {
                    "Gid": {
                        "S": "gidNumber"
                    },
                    "Role": {
                        "S": "comment"
                    },
                    "Uid": {
                        "S": "uidNumber"
                    }
                }
            },
            "ignore_missing_attributes": {
                "BOOL": false
            },
            "port": {
                "N": "636"
            },
            "search_base": {
                "S": "DC=example,DC=com"
            },
            "server": {
                "S": "ldap.example.com"
            },
            "ssl": {
                "BOOL": true
            },
            "ssl_verify": {
                "BOOL": true
            },
            "Role": {
                "S": "arn:aws:iam::123456789012:role/examplecom-AWSTransferRole"
            }
        }
      },
    "module": {
        "S": "ldap"
    }
}
```

### user record

```json
{
    "user": {
        "S": "johnsmith"
    },
    "identity_provider_key": {
        "S": "example.com"
    },
    "config": {
        "M": {
            "HomeDirectoryDetails": {
                "L": [{
                        "M": {
                            "Entry": {
                                "S": "/home"
                            },
                            "Target": {
                                "S": "organization-bucket/users/johnsmith"
                            }
                        }
                    },
                    {
                        "M": {
                            "Entry": {
                                "S": "/finance"
                            },
                            "Target": {
                                "S": "organization-bucket/departments/finance"
                            }
                        }
                    }
                ]
            },
            "HomeDirectoryType": {
                "S": "LOGICAL"
            }
        }
    },
    "ipv4_allow_list": {
        "SS": [
            "172.31.0.0/16",
            "192.168.10.0/24"
        ]
    }
}
```
## Modifying/updating solution parameters
If you need to change the parameters that were used to deploy the solution initially, in most cases you can use modify the installer stack and re-run the deployment pipeline.

1. Go to [*Stacks*](https://console.aws.amazon.com/cloudformation/home#/stacks) in the CloudFormation console and select the solution stack. Click the **Update** button.
2. On the **Update stack** screen, select **Use current template**, then click **Next**. On the **Specify stack details** page, change any parameters needed, then click the **Next** button.
3. On the **Configure stack options** page, click **Next**.
4. At the **Review stack** page, review all parameters and settings, click the checkbox next to *I acknowledge that AWS CloudFormation might create IAM resources with custom names*, then click **Submit**. 
5. Wait for the Cloudformation stack to finish updating, then go to the [*CodePipeline console*](https://console.aws.amazon.com/codesuite/codepipeline/pipelines) and click the **AWSTransferCustomIdP** pipeline to open it. Click **Release change**,  then click **Release** in the dialog that appears.
6. Wait for the pipeline to complete successfully. Once completed, the solution is reconfigured with the updated parameters.

## Uninstall the solution
If you need to uninstall the solution for any reason, you can do so by deleting the both the custom IdP and installer stacks using the steps below.
1. Go to [*Stacks*](https://console.aws.amazon.com/cloudformation/home#/stacks) in the CloudFormation console and select the **[STACK NAME]-awstransfer-custom-idp** stack. Click **Delete**, then click **Delete** again in the dialog that appears. Wait for the deletion to complete.
2. From the [*Stacks*](https://console.aws.amazon.com/cloudformation/home#/stacks) page in the CloudFormation console, select the solution installer stack. Click **Delete**, then click **Delete** again in the dialog that appears. Wait for the deletion to complete.
### Cleanup remaining artifacts
The following resources are not removed during stack deletion and should be manually removed if no longer required.
* The DynamoDB tables used for users and identity providers (`${AWS::StackName}_users` and `${AWS::StackName_identity_providers}`
* The S3 bucket used for SAM artifacts (`aws-sam-cli-managed-default-samclisourcebucket-[hash]`)
* Cloudwatch Log groups for Lambda
* Lambda layer versions used by the custom IdP Lambda

If you deployed the solution with the pipeline installe (`install.yaml`), these items also need to be cleaned up:
* The CodeBuild and CodePipeline artifacts bucket (`${AWS::StackName}-${AWS::AccountId}-${AWS::Region}-artifacts`)
* Cloudwatch Log groups for CodeBuild

## Logging and troubleshooting
The solution includes detailed logging to help with troubleshooting. Below are details on how to configure log levels and use logs for troubleshooting.

### Testing custom identity providers
The AWS Transfer console has a built-in utility to test custom identity providers that use the password authentication method. You can use this to see the output returned when authentication request is made to the custom identity provider, from the viewpoint of the AWS Transfer service. To use the utility, navigate to your [**AWS Transfer Family Servers**](https://console.aws.amazon.com/transfer/servers) in the console, open the details of of the server, and select **Actions > Test** from the upper right corner. 

> [!NOTE]  
> The identity provider tester works with password authentication only. Public key authentication is not supported. We recommend using the identity provier logs, as described below, for further troubleshooting.
>

A successful response will include the session setup details, such as `HomeDirectoryDetails`. Authentication failures or other errors should result in an exception in most cases. An empty response would also indicate an authentication failure. 

The example below shows an authentication failure because of an incorrect password when using the `argon2` module. 

![A screenshot of the identity provider testing console showing a test result](screenshots/ss-troubleshooting-idptest.png)

### Accessing logs
The Lambda function writes all Logs to Cloudwatch Logs, which can be accessed from the [Cloudwatch console](https://console.aws.amazon.com/cloudwatch/home) in the region the solution is deployed in. The name of the log group is `/aws/lambda/${AWS::StackName}_awstransfer_idp`

> [!NOTE]
> For live troubleshooting, consider using [Cloudwatch Live Tail](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogs_LiveTail.html) to view requests in near real-time.

> [!NOTE]  
> If the Lambda logs do not show any failures but authentication still fails and/or prompts for a password, it's possible that AWS Transfer did not successfully verify private key against the public key. It's also possible that required session properties were missing or misconfigured in the user record. Check the [AWS Transfer server log group](https://docs.aws.amazon.com/transfer/latest/userguide/structured-logging.html) for additional details.
>
### Setting the log level
The solution supports two logging levels: `INFO` and `DEBUG`. By default, logging is set to the `INFO` level. While `INFO` logging provides many details about an authentication request, there may be times where it's necessary to retrieve raw request information and values for troubleshooting. To change the log level to `DEBUG` you can do the following:

1. In the CloudFormation console, go to **Stacks** and select the solution stack that was deployed. 
2. Click the **Update** button at the top of the stack list.
3. On the **Prepare template** screen, select **Use existing template** and click **Next**.
4. On the **Specify stack details** screen, change the **LogLevel** setting to `DEBUG` and click **Next**
5. On the **Configure stack options** screen leave all settings as is and click **Next** at the bottom.
6. On the **Review** screen, review all settings, click any of the requires **Capabilities** checkboxes at the bottom, then click **Submit**.
7. Verify the update completes successfully. 

Follow these same steps to return the **LogLevel** setting to `INFO` after finishing troubleshooting.

> [!WARNING]
> Setting the log level to `DEBUG` can result in sensitive information being included in logs, since other Python packages used in the solution will log raw request information. Be mindful of this when sharing these logs and granting access to them. **Consider using `DEBUG` only in non-production environments with non-sensitive test accounts.**

## FAQ
* **Can I connect and use multiple identity providers using the same custom IdP deployment?**
  
  Yes, the solution is designed to support this scenario. To do this, create multiple records in the **identity_providers** DynamoDB table, then define user records associated with those IdPs. 

*  **What happens if I define the same username for multiple IdPs?** 

    If the user specifies the identity provider using the `UserNameDelimiter` when authenticating, that provider will be used. If no identity provider is specified, the identity provider associated with the first user record retrieved will be used for handling authentication. *The solution will not attempt all matching identity providers for the username*. 

* **Can I connect multiple AWS Transfer servers to the same custom IdP deployment?**

  Yes, this is by design so that the same custom IdP solution and identity provider/user records can be used across multiple AWS Transfer servers. One common use case for this is when an organization has separate AWS Transfer servers for both S3 and EFS targets. Both S3 and EFS targets can also be specified in the same user record if `HomeDirectoryType` is set to `LOGICAL` in the record.

* **Can this solution be deployed multiple times in the same AWS account and region?**
  
  Yes, the solution's resources are deployed with name and ARN conventions that avoid collisions. It can be deployed multiple times to support use cases such as multi-tenancy. 

* **Can the same custom IdP deployment be used for AWS Transfer servers that are in multiple regions?**
  
  This can be done only if the API Gateway setting has been enabled and is used as the custom identity provider source.

* **I need to re-deploy the solution, how do I retain my identity provider and user  tables?**
  
  The identity provider and user tables in DynamoDB are retained when the stack is deleted. When re-deploying the solution, reference the table names when creating the stack, and the solution will use the existing tables instead of creating new ones. 

* **Does the AWS Transfer server need to be deployed in the same VPC as the Custom IdP solution?**
  No, it can be deployed independently of the VPC the Custom IdP solution uses. 


## Common issues
* **After deploying the solution, the pipeline fails on the "TestVPCConnectivity" stage.**
  
  This means that the custom IdP Lambda function cannot connect to dependent AWS services such as DynamoDB using the VPC, subnets, and/or security groups specified. Please verify both DynamoDB and any identity providers can be reached from these subnets. 

* **After deploying the solution, authentication requests fail and/or the Lambda function logs show timeouts.**

  Verify the custom IdP solution has been configured to use subnets in a VPC that can reach DynamoDB and IdP targets. Also verify the selected security groups have outbound rules that permit traffic to reach these targets. One way to verify this is to launch an EC2 instance WITHOUT a public IP address in the subnets and attempt to reach the targets. For example, using curl to send a request to the DynamoDB regional endpoint should return an HTTP status code like the one below and not timeout: 
  
  ```
  ➜  ~ curl https://dynamodb.[REGION].amazonaws.com -v
  *   Trying 3.218.182.169:443...
  * Connected to dynamodb.[REGION].amazonaws.com (3.218.182.169) port 443
  * ALPN: curl offers h2,http/1.1
  * (304) (OUT), TLS handshake, Client hello (1):
  *  CAfile: /etc/ssl/cert.pem
  *  CApath: none
  * (304) (IN), TLS handshake, Server hello (2):
  * (304) (IN), TLS handshake, Unknown (8):
  * (304) (IN), TLS handshake, Certificate (11):
  * (304) (IN), TLS handshake, CERT verify (15):
  * (304) (IN), TLS handshake, Finished (20):
  * (304) (OUT), TLS handshake, Finished (20):
  * SSL connection using TLSv1.3 / AEAD-AES256-GCM-SHA384
  * ALPN: server accepted http/1.1
  * Server certificate:
  *  subject: CN=dynamodb.us-east-1.amazonaws.com
  *  start date: Feb  5 00:00:00 2024 GMT
  *  expire date: Feb  3 23:59:59 2025 GMT
  *  subjectAltName: host "dynamodb.us-east-1.amazonaws.com" matched cert's "dynamodb.us-east-1.amazonaws.com"
  *  issuer: C=US; O=Amazon; CN=Amazon RSA 2048 M01
  *  SSL certificate verify ok.
  * using HTTP/1.1
  > GET / HTTP/1.1
  > Host: dynamodb.us-east-1.amazonaws.com
  > User-Agent: curl/8.4.0
  > Accept: */*
  >
  < HTTP/1.1 200 OK
  < Server: Server
  < Date: Wed, 27 Mar 2024 19:48:28 GMT
  < Content-Type: text/plain
  < Content-Length: 42
  < Connection: keep-alive
  < x-amzn-RequestId: ACKLHF5MV4GRO07E4GDI2U5FF3VV4KQNSO5AEMVJF66Q9ASUAAJG
  < x-amz-crc32: 3128867991
  <
  * Connection #0 to host dynamodb.us-east-1.amazonaws.com left intact
  healthy: dynamodb.us-east-1.amazonaws.com %
  ``` 


* 
## Tutorials

### Setting up Okta
Authenticating users with Okta can be as simple as defining an identity provider with the `okta` module and including the `okta_domain` setting. The basic steps for this are as follows:

1. Determine your Okta domain. This should be in the format of `{domain}.okta.com`
2. In the `identity_providers` DynamoDB table, create a new record, replacing `{provider}` with your desired provider name and `{okta_domain}` with your own Okta domain.

  ```json
    {
      "provider": {
        "S": "{provider}"
      },
      "config": {
        "M": {
          "okta_domain": {
            "S": "{okta_domain}"
          }
        }
      },
      "module": {
        "S": "okta"
      }
    }
  ```

3. In the `users` DynamoDB table, create a new record similar to the one below, replacing any placeholders `{}` with real values. Ensure `{username}` matches a valid username in Okta, and `{provider}` is the name of the provider from the previous step.

   ```json
    {
      "user": {
        "S": "{username}"
      },
      "identity_provider_key": {
        "S": "{provider}"
      },
      "config": {
        "M": {
          "HomeDirectoryDetails": {
            "L": [
              {
                "M": {
                  "Entry": {
                    "S": "{virtual path}"
                  },
                  "Target": {
                    "S": "{[bucketname/prefix/to/files}"
                  }
                }
              }
            ]
          },
          "HomeDirectoryType": {
            "S": "LOGICAL"
          },
          "Role": {
            "S": "{arn:aws:iam::[AWS Account Id]:role/[Role Name]}"
          }
        }
      }
    }
  ```

4. Test the identity provider, either by attempting to connect with an SFTP client, or by going to your AWS Transfer Server in the AWS Console and selecting **Actions > Test** in the upper right corner. If you encounter any failures, see the [Troubleshooting section](#logging-and-troubleshooting) for guidance on how to use logs for identifying the issue.


### Configuring Okta MFA


### Configuring Okta to retrieve session settings from user profile attributes


## Contributing a Module
Want to contribute a module? Please see the [CONTRIBUTING] document for more guidance and standards for building a module and contributing it to this solution. [TODO]


## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

