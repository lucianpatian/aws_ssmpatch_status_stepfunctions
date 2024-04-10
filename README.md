Use AWS StepFunctions for SSM Patching Alerts

AWS Step Functions is a service that doesn't require a server to run. It allows you to connect with Lambda functions and other services to construct important business applications.

The service is built around the principle of linked tasks put together in a workflow called state machines. A task possesses the capability to invoke other AWS services or more recently third-party APIs.

Using SSM to handle patch management for EC2 instances makes things easier. However, there are times when you don't want to restart them automatically, especially when they're running critical services like databases. In this case, you prefer to choose when to restart.

To keep track of the EC2 instances that aren't fully compliant in the SSM patch report, you need alerts. My goal was to send alerts to a Microsoft Teams channel, listing the EC2s that aren't compliant and need additional actions like rebooting. Initially, I used a Lambda function to do this but I didn't want to manage its dependencies over time so I switched to using Step Functions, taking advantage of its new feature that supports HTTPS endpoints.

Overview of the State Machine

![](https://res.cloudinary.com/practicaldev/image/fetch/s--KReQqLzf--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_66%2Cw_800/https://github.com/lucianpatian/aws_ssmpatch_status_stepfunctions/blob/main/ssm_step_function.gif%3Fraw%3Dtrue)

The state machine starts with listing all the running EC2 instances in the AWS account. Then it extracts all the instance IDs and filters them based on the ComplianceType=Patch and Status=NON_COMPLIANT parameters.

Next we need to see if there are instances that require our attention or if we jump to the end of the state machine and stop. To achieve this, we use a task that calculates the length of the array from the previous step. If the array is grater then 0 then it will continue to filter the found instances by the tags to format the message sent the Microsoft Teams channel which includes the EC2 names and their IDs.

In the end we call the 3rd party API (arn:aws:states:::http:invoke) to send the formatted message to the Microsoft Teams channel.

The entire process is initiated by an EventBridge rule, which functions based on a cron job. This rule targets the Step Functions state machine that was previously created.

Additional Configuration Details
This state machine can be used to send the message to any communications tool like Slack, MS Teams or to a ticketing system.

For MS Teams, the endpoint URL needs to be encoded so the "@" needs to be replaced with "%40" or you can use a URL shortener service.

An HTTP Task requires an EventBridge connection, which securely manages the authentication credentials of an API provider. A connection specifies the authorization type and credentials to use for authorizing a third-party API.

In our case we are just sending a message/payload to an external URL without the need of authentication but in order to use the StepFunction HTTP Task, we need to create this connection. When creating the connection, as a requirement, you also create an AWS Secret used for authentication. Again, since there's no need to authenticate to the MS Teams channel, the Secret values contain the keyname of the API and the secret is the ARN of the EventBrdige API connection:

![](https://media.dev.to/cdn-cgi/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Fo8ksbof2qtbzqhsjxp4e.png)

![](https://media.dev.to/cdn-cgi/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Fab6md5n9pji7i6faggkt.png)

![](https://media.dev.to/cdn-cgi/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2F0vb5v4j4i30yfza1r25c.png)

Infrastructure as Code
The entire PoC was done using the AWS Console but since we are living in the age of automation, I wanted to have an easy and repeatable way of deploying the solution.

In the past weeks, the Cloudformation service team announced the new IaC generator (infrastructure as code generator) which must be one of the most desired features for years now so I definitely wanted to give it a try.

It turns out that I was able to get the Cloudformation template for all the needed resources pretty easy. The hardest thing was to select from a huge dropdown list all the resources involved in my scenario and to make sure I don't leave out any. After the template was generated it was a bit difficult to find the correct syntax for replacing the hardcoded values with parameters inside the StepFunction JSON definition which now seems like a piece of cake.

If you want to use this solution, create a new stack using the stack.yml file from this repository. This will create the State Machine, an EventBridge trigger using a cron schedule for each Monday at 08:00, a Secret with a dummy value, an EventBridge Connection and all the needed policies/roles needed for the services to interact with each other.
