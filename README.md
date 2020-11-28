# nested

reffer 

to https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-nested-stacks.html

https://aws.amazon.com/blogs/devops/integrating-aws-cloudformation-guard/

The syntax of AWS::CloudFormation::Stack is

{
   "Type" : "AWS::CloudFormation::Stack",
   "Properties" : {
     "NotificationARNs" : [ String, ... ],
     "Parameters" : { CloudFormation Stack Parameters Property Type },
     "TemplateURL" : String,
     "TimeoutInMinutes" : String
   }
}



Nested stacks also allowed him to pass any parameters declared in his master template to child templates by using the Parameters properties as shown above. It also allowed him to reference outputs of one child stack as a parameter in another child stack by using Fn::GetAtt intrinsic function. For example:

{
   "AWSTemplateFormatVersion": "2010-09-09",
   "Resources": {
       "ChildStack01": {
           "Type": "AWS::CloudFormation::Stack",
           "Properties": {
               "TemplateURL": "https://s3.amazonaws.com/cloudformation-templates-us-east-1/VPC.template",
               "TimeoutInMinutes": "60"
           }
       },
       "ChildStack02": {
           "Type": "AWS::CloudFormation::Stack",
           "Properties": {
               "TemplateURL": "https://s3.amazonaws.com/cloudformation-templates-us-east-1/Subnet.template",
               "Parameters": {
                  "VpcId" : { "Fn::GetAtt" : [ "ChildStack01", "Outputs.VpcID" ] },
               },
               "TimeoutInMinutes": "60"
           }
       }
   },
   "Outputs": {
       "StackRef": {
           "Value": { "Ref": "ChildStack02" }
       },
       "OutputFromNestedStack": {
           "Value": { "Fn::GetAtt": [ "ChildStack02", "Outputs.SubnetID" ]}
       }
   }
}

## standard template we can user


AWSTemplateFormatVersion: 
Description: >-
  Creates Pipeline and required resources for TaskCat CI. License: Apache 2.0
  (Please do not remove)(qs-1ops82lkf)
Resources:
  CopyLambdasStack:
    Type: 'AWS::CloudFormation::Stack'
    Properties:
      TemplateURL:
        !Sub
          - 'https://${S3Bucket}.s3.${S3Region}.${AWS::URLSuffix}/${QSS3KeyPrefix}templates/copy-lambdas.template'
          - S3Region: !If [UsingDefaultBucket, !Ref 'AWS::Region', !Ref QSS3BucketRegion]
            S3Bucket: !If [UsingDefaultBucket, !Sub '${QSS3BucketName}-${AWS::Region}', !Ref QSS3BucketName]
      Parameters:
        BucketName: !Ref ArtifactBucket
        QSS3BucketName: !Ref QSS3BucketName
        QSS3BucketRegion: !Ref QSS3BucketRegion
        QSS3KeyPrefix: !Ref QSS3KeyPrefix
  SSMStack:
    Type: 'AWS::CloudFormation::Stack'
    DependsOn:
      - CopyLambdasStack
    Properties:
      TemplateURL:
        !Sub
          - 'https://${S3Bucket}.s3.${S3Region}.${AWS::URLSuffix}/${QSS3KeyPrefix}templates/ssm.template'
          - S3Region: !If [UsingDefaultBucket, !Ref 'AWS::Region', !Ref QSS3BucketRegion]
            S3Bucket: !If [UsingDefaultBucket, !Sub '${QSS3BucketName}-${AWS::Region}', !Ref QSS3BucketName]
      Parameters:
        GithubToken: !Ref GitHubOAuthToken
        LambdaZipsBucket: !GetAtt 
          - CopyLambdasStack
          - Outputs.LambdaZipsBucket
        QSS3KeyPrefix: !Ref QSS3KeyPrefix
  GitMergeLambda:
    Type: 'AWS::Lambda::Function'
    DependsOn:
      - GitMergeRole
      - CopyLambdasStack
      - SSMStack
    Properties:
      Code:
        S3Bucket: !GetAtt 
          - CopyLambdasStack
          - Outputs.LambdaZipsBucket
        S3Key: !Sub '${QSS3KeyPrefix}functions/package/lambda.zip'
      Description: Merge github branches
      FunctionName: Git_Merge
      Handler: !Join 
        - .
        - - git_merge
          - lambda_handler
      Role: !GetAtt 
        - GitMergeRole
        - Arn
      Runtime: python3.6
      Timeout: 30
  GitMergeRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: 'sts:AssumeRole'
      Path: /
      Policies:
        - PolicyName: GitMergeRolePolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'logs:CreateLogGroup'
                  - 'logs:CreateLogStream'
                  - 'logs:PutLogEvents'
                Resource:
                  - !Join 
                    - ''
                    - - 'arn:aws:logs:'
                      - !Ref 'AWS::Region'
                      - ':'
                      - !Ref 'AWS::AccountId'
                      - ':log-group:/aws/lambda/*'
              - Effect: Allow
                Action:
                  - 'codepipeline:GetPipeline'
                  - 'codepipeline:GetPipelineExecution'
                  - 'codepipeline:GetPipelineState'
                  - 'codepipeline:ListPipelines'
                  - 'codepipeline:ListPipelineExecutions'
                Resource:
                  - !Sub 'arn:aws:codepipeline:${AWS::Region}:${AWS::AccountId}:*'
              - Effect: Allow
                Action:
                  - 'codepipeline:GetJobDetails'
                  - 'codepipeline:PutJobSuccessResult'
                  - 'codepipeline:PutJobFailureResult'
                Resource: '*'
              - Effect: Allow
                Action:
                  - 's3:GetObject'
                Resource: '*'
              - Effect: Allow
                Action:
                  - 'ssm:Describe*'
                  - 'ssm:Get*'
                  - 'ssm:List*'
                Resource: '*'
              - Effect: Allow
                Action:
                  - 'kms:Decrypt'
                Resource: !GetAtt 
                  - SSMStack
                  - Outputs.KMSKeyArn
  ArtifactBucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      AccessControl: Private
      LifecycleConfiguration:
        Rules:
          - NoncurrentVersionExpirationInDays: 30
            Status: Enabled
      VersioningConfiguration:
        Status: Enabled
  IAMRoleStack:
    Type: 'AWS::CloudFormation::Stack'
    DependsOn:
      - ArtifactBucket
    Properties:
      TemplateURL:
        !Sub
          - 'https://${S3Bucket}.s3.${S3Region}.${AWS::URLSuffix}/${QSS3KeyPrefix}templates/iam-roles.template'
          - S3Region: !If [UsingDefaultBucket, !Ref 'AWS::Region', !Ref QSS3BucketRegion]
            S3Bucket: !If [UsingDefaultBucket, !Sub '${QSS3BucketName}-${AWS::Region}', !Ref QSS3BucketName]
      Parameters:
        ArtifactBucket: !Ref ArtifactBucket
  CodePipelineStack:
    Type: 'AWS::CloudFormation::Stack'
    DependsOn:
      - ArtifactBucket
      - IAMRoleStack
    Properties:
      TemplateURL:
        !Sub
          - 'https://${S3Bucket}.s3.${S3Region}.${AWS::URLSuffix}/${QSS3KeyPrefix}templates/pipeline.template'
          - S3Region: !If [UsingDefaultBucket, !Ref 'AWS::Region', !Ref QSS3BucketRegion]
            S3Bucket: !If [UsingDefaultBucket, !Sub '${QSS3BucketName}-${AWS::Region}', !Ref QSS3BucketName]
      Parameters:
        GitHubUser: !Ref GitHubUser
        GitHubRepoName: !Ref GitHubRepoName
        SourceRepoBranch: !Ref SourceRepoBranch
        ReleaseBranch: !Ref ReleaseBranch
        GitHubOAuthToken: !Ref GitHubOAuthToken
        ArtifactBucket: !Ref ArtifactBucket
        GitMergeLambda: !Ref GitMergeLambda
        CodePipelineRoleArn: !GetAtt 
          - IAMRoleStack
          - Outputs.CodePipelineRoleArn
        CodeBuildRoleArn: !GetAtt 
          - IAMRoleStack
          - Outputs.CodeBuildRoleArn
Parameters:
  GitHubUser:
    Description: Enter GitHub username of the repository owner
    Type: String
  GitHubRepoName:
    Description: Enter the repository name that should be monitored for changes
    Type: String
  SourceRepoBranch:
    Description: Enter the branch name to be monitored
    Type: String
  ReleaseBranch:
    Description: >-
      Enter the release branch name. On successfull build, above branch will be
      merged into this branch.
    Type: String
  GitHubOAuthToken:
    Description: >-
      Create a token with 'repo' and 'admin:repo_hook' permissions here
      https://github.com/settings/tokens
    Type: String
    NoEcho: 'true'
  QSS3BucketName:
    AllowedPattern: '^[0-9a-zA-Z]+([0-9a-zA-Z-]*[0-9a-zA-Z])*$'
    ConstraintDescription: >-
      Quick Start bucket name can include numbers, lowercase letters, uppercase
      letters, and hyphens (-). It cannot start or end with a hyphen (-).
    Default: aws-quickstart
    Description: >-
      S3 bucket name for the Quick Start assets. Quick Start bucket name can
      include numbers, lowercase letters, uppercase letters, and hyphens (-). It
      cannot start or end with a hyphen (-).
    Type: String
  QSS3BucketRegion:
    Default: 'us-east-1'
    Description: 'The AWS Region where the Quick Start S3 bucket (QSS3BucketName) is hosted. When using your own bucket, you must specify this value.'
    Type: String
  QSS3KeyPrefix:
    AllowedPattern: '^[0-9a-zA-Z-/]*$'
    ConstraintDescription: >-
      Quick Start key prefix can include numbers, lowercase letters, uppercase
      letters, hyphens (-), and forward slash (/).
    Default: quickstart-taskcat-ci/
    Description: >-
      S3 key prefix for the Quick Start assets. Quick Start key prefix can
      include numbers, lowercase letters, uppercase letters, hyphens (-), and
      forward slash (/).
    Type: String
Metadata:
  'AWS::CloudFormation::Interface':
    ParameterGroups:
      - Label:
          default: GitHub Configuration
        Parameters:
          - GitHubUser
          - GitHubRepoName
          - SourceRepoBranch
          - ReleaseBranch
          - GitHubOAuthToken
      - Label:
          default: AWS Quick Start Configuration
        Parameters:
          - QSS3BucketName
          - QSS3BucketRegion
          - QSS3KeyPrefix
    ParameterLabels:
      GitHubUser:
        default: Repository owner
      GitHubRepoName:
        default: Repository name
      SourceRepoBranch:
        default: Source branch
      ReleaseBranch:
        default: Release branch
      GitHubOAuthToken:
        default: OAuth2 token
      QSS3BucketName:
        default: Quick Start S3 Bucket Name
      QSS3BucketRegion:
        default: Quick Start S3 bucket region
      QSS3KeyPrefix:
        default: Quick Start S3 Key Prefix

Conditions:
  UsingDefaultBucket: !Equals [!Ref QSS3BucketName, 'aws-quickstart']

Outputs:
  CodePipelineURL:
    Description: The URL of the created Pipeline
    Value: !Sub 
      - >-
        https://${AWS::Region}.console.aws.amazon.com/codepipeline/home?region=${AWS::Region}#/view/${CodePipelineName}
      - CodePipelineName: !GetAtt 
          - CodePipelineStack
          - Outputs.CodePipelineName
  TaskCatReports:
    Description: Path to the TaskCat report. Each report is named as CODEBUILD_BUILD_ID.zip
    Value: !Sub 's3://${ArtifactBucket}/taskcat_reports/'
