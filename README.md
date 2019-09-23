# AWS CloudFormation Pipeline Example

This repository was created as part of a blog post I wrote for automated testing for CloudFormation templates. The full
post can be found [here](https://www.sep.com/sep-blog/automated-testing-for-cloudformation-templates).

## Testing Tools

### ValidateTemplate

This is the most basic test you can run on a CloudFormation template. `ValidateTemplate` is an action available on the
CloudFormation API and will determine if the file provided is a valid JSON/YAML CloudFormation template. This command
can be called locally using the AWS CLI and does require authentication against an AWS account in order to work successfully.

Test Command:
```sh
aws cloudformation validate-template --template-body file://templates/workload.template
```

Documentation: https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_ValidateTemplate.html

### yamllint

Linting isn't necessarily a testing tool, but it is a great tool that allows for keeping code formatted in a ~~clean~~
consistent way. Having a tool like this run on every commit ~~eliminates~~ limits the number of code review comments
related to style, allowing developers to focus in on the substance of the change instead of the style. yamllint is
available as a Python module and supports configuring of rules from a configuration file as well as disabling checks
via comments.

Test Command:
```sh
yamllint --strict templates/workload.template
```

Documentation: https://yamllint.readthedocs.io/en/stable/configuration.html

### cfn-lint

Another linter, but this one goes beyond basic style checking. The CloudFormation Linter is an open-source project currently
maintained by the CloudFormation team and has been around since April 2018. It aims to check your CloudFormation template
against the CloudFormation specification. Additionally, it will detect things like incorrect values for resource properties
as well as looking for unused parameters. It also supports some basic customization that allows you to tweak the rules to
meet the specific needs of your project. Similar to yamllint, this tool is available as a Python module.

Test Command:
```sh
cfn-lint --include-checks I --template templates/workload.template
```

Documentation: https://github.com/aws-cloudformation/cfn-python-lint

### cfn-nag

Yet another linting tool, but this one looks at your template to ensure best practices are being followed. cfn-nag will find
things like wildcards in IAM policies or S3 buckets that don't have encryption enabled by default. This is an open-source
project maintained by stelligent and is available as a Ruby gem and a Docker image.

Test Command:
```sh
cfn_nag --fail-on-warnings --output-format txt templates/workload.template
```

Documentation: https://github.com/stelligent/cfn_nag

### taskcat

Another linting tool (just kidding). taskcat allows for testing your CloudFormation templates by performing test deployments
in multiple regions and providing the results for each deployment. The tool can be configured to run multiple different
test scenarios with different input parameters to fully exercise your templates. It also has the ability to generate values
for your parameters that helps eliminiate some sticky parameters in your template that have to be globally unique (e.g.
BucketName). taskcat is another open-source tool maintained by the folks at AWS and is available as a Python module and
a Docker image.

> Note: In my experience, this tool doesn't work on Windows, so I would highly recommend using the Docker image if that is
your development operating system of choice.

Test Command:
```sh
taskcat -c ci/taskcat.yml | sed "s,\x1B\[[0-9;]*[a-zA-Z],,g"
TASKCAT_RESULT=${PIPESTATUS[0]}

ls -1 taskcat_outputs/*.txt | while read LOG
do
  echo
  echo "> $LOG"
  cat ${LOG}
done

exit $TASKCAT_RESULT
```

I know this one looks a bit scary, so let's break it down. Unfortunately running taskcat on CodeBuild has two flaws
(color escape codes, and not printing logs to STDOUT). The command(s) here will run taskcat and strip the color escape
codes from the output. It then saves off the exit code from taskcat to `TASKCAT_RESULT`. Next, it will loop over all of
the log files in the `taskcat_outputs/` folder and will print the filename and the contents of the file to STDOUT. Finally,
it will exit with the exit code from taskcat so that if the taskcat tests fail, the test stage will fail, ultimately stopping
a deployment to production if an error was discovered.

Documentation: https://github.com/aws-quickstart/taskcat

## Template Highlights

### Pipeline

The pipeline itself. Arguably the most important resource in the template (it is in the name). This resource controls the
steps that are taken, from checkout, to test, and ultimately deploy.

```yaml
Pipeline:
  Type: AWS::CodePipeline::Pipeline
  Properties:
    ...
    Stages:
      - Name: Source
        ...
      - Name: Test
        ...
      - Name: Deploy
        ...
```

While I won't go into the details of how each of these stages are configured, this resource controls the steps of the
pipeline and how they are run. The [documentation for the Pipeline resource type][aws-codepipeline-pipeline-resource-type-documentation]
is a great source of inspiration to determine what all is possible. Additionally, the
[documentation for the configuration of the various actions][codepipeline-action-configuration-documentation]
is also helpful to figure out what is required for each action within a given stage.

### Pipeline Webhook

This is almost a requirement nowadays for a CI/CD tool. By supporting webhooks from GitHub, the pipeline can be triggered
as soon as your code is committed to the repository. This ensures that changes can work their way through the pipeline
as quickly as possible and reduce the time it takes to get a change to production. This resource requires the `admin:repo_hook`
permission for your GitHub personal access token.

```yaml
PipelineWebhook:
  Type: AWS::CodePipeline::Webhook
  Properties:
    Authentication: GITHUB_HMAC
    AuthenticationConfiguration:
      SecretToken: !Sub ${GitHubPersonalAccessToken}
    RegisterWithThirdParty: true
    Filters:
      - JsonPath: $.ref
        MatchEquals: refs/heads/{Branch}
    TargetPipeline: !Sub ${Pipeline}
    TargetAction: Source
    TargetPipelineVersion: !Sub ${Pipeline.Version}
```

### CodeBuild PR Project

I didn't really cover this previously, but the template also includes a CodeBuild project to run the same `buildspec.yml`
file on pull requests as well. This can give you earlier feedback on the test results before merging the changes into `master`.
CodeBuild is also capable of reporting the status of the tests back to GitHub, so the results of these checks can be viewed
directly from the pull request without having to log into AWS to determine if the tests are passing or failing.

```yaml
CodeBuildPRProject:
  Type: AWS::CodeBuild::Project
  Properties:
    ...
    Name: !Sub ${GitHubRepositoryName}-pull-requests
    ServiceRole: !Sub ${CodeBuildRole.Arn}
    Source:
      GitCloneDepth: 1
      Location: !Sub "https://github.com/${GitHubRepositoryOwner}/${GitHubRepositoryName}.git"
      ReportBuildStatus: true
      Type: GITHUB
    Triggers:
      Webhook: true
      FilterGroups:
        - - Type: EVENT
            Pattern: PULL_REQUEST_CREATED, PULL_REQUEST_UPDATED, PULL_REQUEST_REOPENED
            ExcludeMatchedPattern: false
          - Type: BASE_REF
            Pattern: !Sub ^refs/heads/${GitHubIntegrationBranch}$
            ExcludeMatchedPattern: false
```

### CloudFormation Stack Policy

You can't get through any blog post about AWS without talking about IAM at some point. While most of the roles and policies
in this template shouldn't need to change, the `CloudFormationStackPolicy` resource definitely will to meet your purposes.
This policy defines the permissions granted to CloudFormation when deploying the `templates/workload.template` to production.
It is important to remember that this role will stay attach to the stack and be used on subsequent updates (unless replaced
with a different role). This means the permissions granted to CloudFormation also need to consider what permissions are
required to delete the stack. Any time you modify the workload template, make sure to update this policy to include permissions
to cover any new resource types or configuration that is being added.

```yaml
CloudFormationStackPolicy:
  Type: AWS::IAM::ManagedPolicy
  Properties:
    PolicyDocument:
      Version: "2012-10-17"
      Statement:
        - Effect: Allow
          Action:
            - s3:CreateBucket
            - s3:DeleteBucket
            - s3:GetEncryptionConfiguration
            - s3:PutEncryptionConfiguration
            - s3:GetBucketAcl
            - s3:PutBucketAcl
            - s3:GetBucketLogging
            - s3:PutBucketLogging
          Resource: arn:aws:s3:::*
```