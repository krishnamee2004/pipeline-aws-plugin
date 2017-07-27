# Features

This plugins adds Jenkins pipeline steps to interact with the AWS API.

* Invalidating CloudFront distributions
* Creating, updating and deleting CloudFormation stacks
* Up- and downloading files to/from S3

[**see the changelog for release information**](#changelog)

# Usage / Steps

## withAWS

the `withAWS` step provides authorization for the nested steps.
You can provide region and profile information or let Jenkins
assume a role in another or the same AWS account.
You can mix all parameters in one `withAWS` block.

Set region information:

```
withAWS(region:'eu-west-1') {
    // do something
}
```

Use Jenkins UsernamePassword credentials information (Username: AccessKeyId, Password: SecretAccessKey):

```
withAWS(credentials:'nameOfSystemCredentials') {
    // do something
}
```

Use profile information from `~/.aws/config`:

```
withAWS(profile:'myProfile') {
    // do something
}
```

Assume role information (account is optional - uses current account as default, externalId is optional):

```
withAWS(role:'admin', roleAccount:'123456789012', externalId: 'my-external-id') {
    // do something
}
```

## awsIdentity

Print current AWS identity information to the log.

```
awsIdentity()
```

## cfInvalidate

Invalidate given paths in CloudFront distribution.

```
cfInvalidate(distribution:'someDistributionId', paths:['/*'])
```

## s3Upload

Upload a file/folder from the workspace to an S3 bucket.
If the `file` parameter denotes a directory, the complete directory including all subfolders will be uploaded.

```
s3Upload(file:'file.txt', bucket:'my-bucket', path:'path/to/target/file.txt')
s3Upload(file:'someFolder', bucket:'my-bucket', path:'path/to/targetFolder/')
```

## s3Download

Download a file/folder from S3 to the local workspace.
Set optional parameter `force` to `true` to overwrite existing file in workspace.
If the `path` ends with a `/` the complete virtual directory will be downloaded.

```
s3Download(file:'file.txt', bucket:'my-bucket', path:'path/to/source/file.txt', force:true)
s3Download(file:'targetFolder/', bucket:'my-bucket', path:'path/to/sourceFolder/', force:true)
```

## s3Delete

Delete a file/folder from S3.
If the path ends in a "/", then the path will be interpreted to be a folder, and all of its contents will be removed.

```
s3Delete(bucket:'my-bucket', path:'path/to/source/file.txt')
s3Delete(bucket:'my-bucket', path:'path/to/sourceFolder/')
```

## s3FindFiles

This provides a way to query the files/folders in the S3 bucket, analogous to the `findFiles` step provided by "pipeline-utility-steps-plugin".
If specified, the `path` limits the scope of the operation to that folder only.
The `glob` parameter tells `s3FindFiles` what to look for.  This can be a file name, a full path to a file, or a standard glob ("\*", "\*.ext", "path/\*\*/file.ext", etc.).

If you do not specify `path`, then it will default to the root of the bucket.
The path is assumed to be a folder; you do not need to end it with a "/", but it is okay if you do.
The `path` property of the results will be _relative_ to this value.

This works by enumerating _every_ file/folder in the S3 bucket under `path` and then performing glob matching.
When possible, you should use `path` to limit the search space for efficiency purposes.

If you do not specify `glob`, then it will default to "\*".

By default, this will return both files and folders.
To only return files, set the `onlyFiles` parameter to `true`.

```
files = s3FindFiles(bucket:'my-bucket')
files = s3FindFiles(bucket:'my-bucket', glob:'path/to/targetFolder/file.ext')
files = s3FindFiles(bucket:'my-bucket', path:'path/to/targetFolder/', glob:'file.ext')
files = s3FindFiles(bucket:'my-bucket', path:'path/to/targetFolder/', glob:'*.ext')
files = s3FindFiles(bucket:'my-bucket', path:'path/', glob:'**/file.ext')
```

`s3FindFiles` returns an array of `FileWrapper` objects exactly identical to those returned by `findFiles`.

Each `FileWrapper` object has the following properties:

* `name`: the filename portion of the path (for "path/to/my/file.ext", this would be "file.ext")
* `path`: the full path of the file, _relative_ to the `path` specified (for `path`="path/to/", this property of the file "path/to/my/file.ext" would be "my/file.ext")
* `directory`: true if this is a directory; false otherwise
* `length`: the length of the file (this is always "0" for directories)
* `lastModified`: the last modification timestamp, in milliseconds since the Unix epoch (this is always "0" for directories)

When used in a string context, a `FileWrapper` object returns the value of its `path` property.

## cfnValidate

Validates the given CloudFormation template.

```
cfnValidate(file:'template.yaml')
```

## cfnUpdate

Create or update the given CloudFormation stack using the given template from the workspace.
You can specify an optional list of parameters.
You can also specify a list of `keepParams` of parameters which will use the previous value on stack updates.

Using `timeoutInMinutes` you can specify the amount of time that can pass before the stack status becomes CREATE_FAILED and the stack gets rolled back.
Due to limitations in the AWS API, this only applies to stack creation.

If you have many parameters you can specify a `paramsFile` containing the parameters. The format is either a standard
JSON file like with the cli or a YAML file for the [cfn-params](https://www.npmjs.com/package/cfn-params) command line utility.

Additionally you can specify a list of tags that are set on the stack and all resources created by CloudFormation.
The step returns the outputs of the stack as a map.

To prevent running into rate limiting on the AWS API you can change the default polling interval of 1000 ms using the parameter `pollIntervall`. Using the value `0` disables event printing.

```
def outputs = cfnUpdate(stack:'my-stack', file:'template.yaml', params:['InstanceType=t2.nano'], keepParams:['Version'], timeoutInMinutes:10, tags:['TagName=Value'], pollInterval:1000)
```

Alternatively, you can specify a URL to a template on S3 (you'll need this if you hit the 51200 byte limit on template):

```
def outputs = cfnUpdate(stack:'my-stack', url:'https://s3.amazonaws.com/my-templates-bucket/template.yaml')
```

Note: When creating a stack, either `file` or `url` are required. When updating it, omitting both parameters will keep the stack's current template.

## cfnDelete

Remove the given stack from CloudFormation.

To prevent running into rate limiting on the AWS API you can change the default polling interval of 1000 ms using the parameter `pollIntervall`. Using the value `0` disables event printing.

```
cfnDelete(stack:'my-stack', pollInterval:1000)
```

## cfnDescribe

The step returns the outputs of the stack as map.

```
def outputs = cfnDescribe(stack:'my-stack')
```

## cfnExports

The step returns the global CloudFormation exports as map.

```
def globalExports = cfnExports()
```

## snsPublish

Publishes a message to SNS.

```
snsPublish(topicArn:'arn:aws:sns:us-east-1:123456789012:MyNewTopic', subject:'my subject', message:'this is your message')
```

## deployAPI

Deploys an API Gateway definition to a stage.

```
deployAPI(api:'myApiId', stage:'Prod')
```

Additionally you can specify a description and stage variables.

```
deployAPI(api:'myApiId', stage:'Prod', description:"Build: ${env.BUILD_ID}", variables:['key=value'])
```

## awaitDeploymentCompletion

Awaits for a CodeDeploy deployment to complete.

The step runs within the `withAWS` block and requires only one parameter:

* deploymentId (the AWS CodeDeploy deployment id: e.g. 'd-3GR0HQLDN')

Simple await:
```
awaitDeploymentCompletion('d-3GR0HQLDN')
```

Timed await:
```
timeout(time: 15, unit: 'MINUTES'){
    awaitDeploymentCompletion('d-3GR0HQLDN')
}
```

## listAWSAccounts

Retrieves the list of all AWS accounts of the organization. This step can only be run in the master account.

The step returns an array of Account objects with the following fields:

* id
* arn
* name
* status

```
def accounts = listAWSAccounts()
```

# Changelog

## 1.13 (master)
* Add `s3FindFiles` step

## 1.12
* Make polling interval for CFN events configurable #JENKINS-45348
* Add `awaitDeploymentCompletion` step
* Add `s3Delete` step
* Add `listAWSAccounts` step

## 1.11
* Replace slash in RoleSessionName coming from Job folders

## 1.10
* improve S3 download logging #JENKINS-44903
* change RoleSessionName to include job name and build number
* add the ability to use a URL in cfnValidate

## 1.9
* add support for create stack timeout
* add the ability to use a URL in cfnUpdate
* add deployAPI step

## 1.8
* add support for externalId for role changes
* allow path to be null or empty in S3 steps

## 1.7
* fix environment for withAWS step
* add support for recursive S3 upload/download

## 1.6
* fix #JENKINS-42415 causing S3 errors on slaves
* add paramsFile support for cfnUpdate
* allow the use of Jenkins credentials for AWS access #JENKINS-41261

## 1.5
* add cfnExports step
* add cfnValidate step
* change how s3Upload works to use the aws client to guess the correct content type for the file.

## 1.4
* add empty checks for mandatory strings
* use latest AWS SDK
* add support for CloudFormation stack tags

## 1.3
* add support for publishing messages to SNS
* fail step on errors during CloudFormation actions

## 1.2
* add proxy support using standard environment variables
* add cfnDescribe step to fetch stack outputs

## 1.1
* fixing invalidation of CloudFront distributions
* add output of stack creation, updates and deletes
* Only fetch AWS environment once
* make long-running steps async

## 1.0
* first release containing multiple pipeline steps


### Issue:
* When we invoke ListAccountsRequest(), AWS returns only 20 accounts and a token. In our case, we have 50+ accounts. Would you be able to update the code to support token?
* I was unable to ask a question and hence this PR.
