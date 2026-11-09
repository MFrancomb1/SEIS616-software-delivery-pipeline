# SEIS616-software-delivery-pipeline

# SEIS 616 — Software Delivery Pipeline with Manual Approval

The `ManualApproval` stage is placed **after Build and before Deploy**. The Deploy stage cannot run for that execution until an authorized reviewer approves the pending action. Rejecting the action stops that pipeline execution.

## CloudFormation modifications

The template is based on the provided `app-codepipeline-template.json`. The following changes were made:


- Added `ApprovalEmail` parameter
  - Accept the reviewer's email address when the stack is created
- Added `ApprovalNotificationTopic` (`AWS::SNS::Topic`)
  -  Publish manual approval requests
- Added `ApprovalEmailSubscription` (`AWS::SNS::Subscription`)
  -  Deliver approval notifications by email, after subscription confirmation
- Added `ManualApproval` stage and `ReviewAndApprove` action
  -  Pause execution until an authorized user approves or rejects 
- Set the approval action's `NotificationArn`
  -  Connect the approval request to the SNS topic 
- Replaced `AWS::AutoScaling::LaunchConfiguration` with `AWS::EC2::LaunchTemplate`
  -  Address the unsupported legacy resource in the AWS account 
- Updated the artifact bucket policy
  -  Require HTTPS transport while retaining default S3 server-side encryption 
- Set GitHub source polling to `true`
  -  Detect source changes using the template's legacy GitHub source action 
- Updated default repository name
  - Use `seis616-python-project` 

## Deployment parameters

When creating the CloudFormation stack, supply:

- `GitHubUsername`: `MFrancomb1`
- `GitHubRepositoryName`: `seis616-python-project`
- `GitHubBranchName`: `master`
- `projectUrl`: `https://github.com/MFrancomb1/seis616-python-project.git`
- `GitHubAccessToken`: a private GitHub token with appropriate repository access (**never commit it**)
- `ApprovalEmail`: the reviewer's email address
- `KeyName`: an existing EC2 key pair
- `YourIp`: the administrator's public IP address in CIDR format (for example, `203.0.113.10/32`)
- `webami`: a compatible Ubuntu AMI for the chosen AWS region


## Verification report

### 1. Stack Creation

Created the modified CloudFormation stack. The stack reached **`CREATE_COMPLETE`**. An SNS subscription confirmation email arrived at the address supplied through `ApprovalEmail`.


### 3. Automated build and test

The initial build failed because Flask 2.0.3 was incompatible with the newer Werkzeug package. Added `Werkzeug==2.0.3` to the application repository's `requirements.txt`.

The build then encountered a Node.js 14 / LESS compiler compatibility issue. Updated `buildspec.yml` to install a compatible LESS version (`less@4.1.3`). The next pipeline execution advanced to the manual approval stage, meaning application test passed and the pipeline advanced beyond Build after the dependency fixes.

### 4. Manual approval and email notification

The pipeline entered the **`ManualApproval`** stage, with the **`ReviewAndApprove`** action waiting for approval. The **Deploy** stage had not run.

Amazon SNS delivered an email headed **CodePipeline manual approval**, stating that the approval action was waiting for a response.

### 5. Approval

Opened the CodePipeline review dialog, selected **Approve**

After approval, CodeDeploy began a deployment to the two EC2 instances in the Auto Scaling group using `CodeDeployDefault.OneAtATime`.

The approval allowed the pipeline to proceed to the Deploy stage.

### 6. Final deployment verification

CodeDeploy reports success for both instances.
