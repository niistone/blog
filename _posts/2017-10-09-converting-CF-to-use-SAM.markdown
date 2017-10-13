---
layout: post
title:  "Converting a plain ol' CloudFormation template to use SAM"
date:   2017-10-09 15:18:11 -0700
categories: jekyll update
---
Converting an existing "vanilla" CloudFormation template to use AWS Serverless Architecture Model (SAM) is a mostly straightforward process, although there are a few steps that may involve some under or undocumented tweaks. Some of the steps mentioned in this post may also be helpful for building SAM applications that include some CloudFormation resources that may not (yet) exist in SAM.

Here is an [intro][intro] to SAM, the SAM [specification][specification], and a collection of (very basic) [examples][examples].

The following is an intial plan of attack which walks you through the process of conversion and includes a small collection of tweaks and hurdles you may come across. 

Converting to AWS::Serverless:Function
--------------------------------------
### Prep and the package -> deploy process
Start by creating a new template and copying your existing one over. You will want the old template as a reference as you continue on. Start by converting all AWS::Lambda::Function resources over to [AWS::Serverless:Function][AWS::Serverless:Function] resources. This is just a straightforward find and replace. AWS SAM applications use a [package -> deploy process][package -> deploy process] to speed up development and make templates a bit more generic. Essentially, you treat your CF template as a slightly more generic template, as it contains relative links to files containing lambda code. When you run 'aws cloudformation package', those local files get uploaded to an s3 bucket and the relative links in the template are replaced with freshly updated s3 bucket links. The command creates a packaged template that is ready to deploy with all the changed code artifact links.

### Lambda resource tweaks
In order to get this to work, you need to rename all your "Code" properties in your AWS::Serverless::Function resources (or AWS::Lambda::Function if you haven't changed yet) to "CodeUri" so that they are in accordance to the SAM specification. In addition, you need to convert whatever comes after "CodeUri" to be a relative path to the code that is associated with the Lambda function described by the resource. Leave any role resources associated your Lambda functions in the template, SAM does not create these for you. 

You should now be able to package and deploy your template through the [AWS cli][AWS cli]. 

Note: You can still use the [CloudFormation console][CloudFormation console] if you wish, but you must provide a packaged template as input.

Below is an example of what changes you need to make. There isn't too much different between the two resources, mainly a change to the resource's "Type", the "Code" property change to "CodeUri", and the link to the code -- which needs to be a relative path.

{% highlight json %}
"BackingLambdaFunctionLambdaOne": {
	"Type": "AWS::Lambda::Function",
	"Properties": {
	    "Code": "source_for_all_my_lambda_functions/directory_containing_code_of_lambda_one_function" 
	    OR { "S3Bucket": "lambda-bucket", "S3Key": "lambda_one.zip" },
	    "FunctionName": "LambdaOne",
	    "Handler": "lambda_one.lambda_handler",
	    "MemorySize": "128",
	    "Role": { "Fn::GetAtt": [ "BackingLambdaExecutionRoleLambdaOne", "Arn" ] },
	    "Runtime": "python2.7",
	    "Timeout": "90",
	    "Environment": {
	        "Variables": {
	            "region": { "Ref": "AWS::Region" }
	        }
	    }
	}
}
{% endhighlight %}

... Changes to ...

{% highlight json %}
"BackingLambdaFunctionLambdaOne": {
	"Type": "AWS::Serverless::Function",
	"Properties": {
	    "CodeUri": "source_for_all_my_lambda_functions/directory_containing_code_of_lambda_one_function",
	    "FunctionName": "LambdaOne",
	    "Handler": "lambda_one.lambda_handler",
	    "MemorySize": "128",
	    "Role": { "Fn::GetAtt": [ "BackingLambdaExecutionRoleLambdaOne", "Arn" ] },
	    "Runtime": "python2.7",
	    "Timeout": "90",
	    "Environment": {
	        "Variables": {
	            "region": { "Ref": "AWS::Region" }
	        }
	    }
	}
}
{% endhighlight %}
Converting to AWS::Serverless:Api
---------------------------------
Hopefully that was straightforward. The trickier part is refactoring the API Gateway code to use SAM. I would start with a fresh template that just sets up the API gateway and add back the serverless function resources and any other resources not covered by SAM (roles, Step Functions, etc.) once the gateway is set up to your liking. This can start as simple as the below CF template:

{% highlight json %}
{
	"AWSTemplateFormatVersion": "2010-09-09",
	"Transform": "AWS::Serverless-2016-10-31",
	"Resources": {
	    "MyAPI": {
	        "Type": "AWS::Serverless::Api",
	        "Properties": {
	            "DefinitionUri": "s3://apibucket/api_template.json",
	            "Name": "My API",
	            "StageName": "ApiLATEST",
	        }
	    }
	}
}
{% endhighlight %}

Notice that the DefinitionUri property that points to a buckets with some sort of json artifact. That json artifact is actually just a swagger template of API gateway resources that have been manually uploaded to a bucket (After we have a SAM template that builds our gateway correctly we should change the DefintionUri to use a relative filepath so that it fits into the same [package -> deploy process][package -> deploy process] used in our lambda functions). 

### Getting the template
Usually you would need to create this swagger template yourself, however since you already have a working CloudFormation template, you can actually just export the swagger template from your current API Gateway. You can download this template by navigating to the [API Gateway console][API Gateway console], selecting your API, clicking on the Stages option, selecting the stage you wish to export as a template, and clicking the export tab. Export the stage as "Swagger + API Gateway Extensions" in json or yaml. I chose json because I like parentheses.

You will need to remove the host key value pair in the template as this host is associated with the current deployment and will no longer be correct when you redeploy. 

### How it all unfolds
When you do deploy the template, the AWS::Serverless:API resource and associated swagger template will unroll into the following resources, [AWS::ApiGateway::Stage][AWS::ApiGateway::Stage], [AWS::ApiGateway::Deployment][AWS::ApiGateway::Deployment], [AWS::ApiGateway::RestApi][AWS::ApiGateway::RestApi], [AWS::ApiGateway::Resource][AWS::ApiGateway::Resource], and [AWS::ApiGateway::Method][AWS::ApiGateway::Method] (method and resource will only be created). 

### Roles and API key authentication
Any role resources associated with these still need to be described explicitly in the CloudFormation template. If you are using any sort of API Key authentication, you will need to include those resources ([AWS::ApiGateway::ApiKey][AWS::ApiGateway::ApiKey], [AWS::ApiGateway::UsagePlan][AWS::ApiGateway::UsagePlan], [AWS::ApiGateway::UsagePlanKey][AWS::ApiGateway::UsagePlanKey]) in the template, and change any reference to the API stage to the string in the StageName property of the AWS::Serverless::Function resource. 

### Regarding resources with stage dependencies
One important limitation that has no documentation is the fact that any resource that depends on the completion of the stage resource, such as API key usage plans, need to include a DependsOn statement with the Logical ID of the stage. Since the naming convention of this ID is not documented, you may have to first run the template without any of the resources that depend on the completed stage, then find out what the Logical ID of the stage is in the resources tab of the [CloudFormation console][CloudFormation console]. The following CF UsagePlan resource is an example of the changes, the hardcoded stage name and the Logical ID of the stage in the DependsOn.


{% highlight json %}
"usagePlan" : {
    "Type" : "AWS::ApiGateway::UsagePlan",
    "DependsOn" : ["MyAPIApiLATESTStage"],
    "Properties" : {
        "ApiStages" : [ {"ApiId" : { "Ref" : "MyAPI" }, "Stage" : "ApiLATEST"} ],
        "Description" : "Single key usage plan",
        "UsagePlanName" : "My_Plan"
    }
}
{% endhighlight %}

Conclusion
----------

As for the third SAM resource (DynamoDB) that exists, I used an [AWS Step Function][AWS Step Function] to coordinate state, so I don't have any advice for you there!

If you're curious about what a SAM application using Lambda, Step Functions, and API Gateway (with a bit of Simple Email Service sprinkled in for flavor) have a look at the [code][AR code] that spawned this post.

Bonus...
--------

### Tricky things/the errors that that you googled that brought you here

#### invalid function ARN/Cross-account pass role is not allowed
These errors occur when the ARN in your swagger template is invalid. Don't read into the cross-account error too much, its just telling you that the account id being used is wrong. You can't (as of the time this is posted) pass `Account ID` and `Region` as stage variables to the swagger template (through the [AWS::Serverless::Api][AWS::Serverless::Api] `Variables` property. As far as I can see, you will have to settle for hardcoded values in your swagger template. 


[intro]: https://github.com/awslabs/serverless-application-model
[specification]: https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md
[examples]: https://github.com/awslabs/serverless-application-model/tree/master/examples/2016-10-31
[AWS::Serverless::Api]: https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#awsserverlessapi
[AWS Step Function]: https://aws.amazon.com/step-functions/
[AWS::ApiGateway::Stage]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-stage.html
[AWS::ApiGateway::Deployment]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-deployment.html
[AWS::ApiGateway::RestApi]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-restapi.html
[AWS::ApiGateway::Resource]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-resource.html
[AWS::ApiGateway::Method]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-method.html
[AWS::ApiGateway::UsagePlan]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-usageplan.html
[AWS::ApiGateway::UsagePlanKey]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-usageplankey.html
[AWS::ApiGateway::ApiKey]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-apikey.html
[package -> deploy process]: http://docs.aws.amazon.com/lambda/latest/dg/serverless-deploy-wt.html#serverless-deploy
[API Gateway console]: https://aws.amazon.com/api-gateway/
[CloudFormation console]: https://console.aws.amazon.com/cloudformation/
[AWS cli]: https://aws.amazon.com/cli/
[AR code]: https://github.com/splunk/splunk-aws-adaptive-response
