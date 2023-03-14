# Abstract

In this post, we'll demonstrate how you can build and run a solution to solve many-to-many vehicle routing with Amazon Location Services and HERE Tour Planning.
The solution uses Infrastructure as Code (IaC) solution from AWS called the Cloud Development Kit (CDK). This provides a programmatic solution to provisioning and version control of infrastructure. The solution utilizes event driven architecture based on AWS Lambda. It uses Amazon S3 and Amazon DynamoDB to store the artefacts generated by the solution and integrates with Amazon Location Service to plot those routes visually on a map for each delivery driver.

## Prerequisites

1. An AWS Account [how to set up an account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/)
1. Install [Node.js](https://nodejs.org/en/)
1. Install [AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/cli.html)
1. Install [AWS Amplify CLI](https://docs.amplify.aws/cli/)
1. Github Repository access
- git clone https://gitlab.aws.dev/mgmahesh/aws-here-optimize-fleet-utilization.git

1. Install [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions)
- ensure you are [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html) and authenticated with your AWS account with the CLI 
- You may use any IDE such as [Visual Studio Code](https://code.visualstudio.com/)

Optional:

1.	HERE API Key (optional)
- To acquire an API key, create a free account on the Here Platform, then follow the instructions in the Here Tour Planning documentation to create your API key. 
o	The repository also includes a problem and pre-solved solution file, so you don’t need to acquire a HERE API Key to learn about these offerings

### Deploy Stack with HERE API KEY
For this, we will use a node environment variable. Never commit the actual HERE_API_KEY to source control! You can specify the following:

    HERE_API_KEY=<here-api-key-value> cdk deploy

> For best security practices, add AWS Secret Manager to store the HERE_API_KEY and retrieve securely from there that resource.

## Solution overview

This setup will walk you through how to generate the infrastructure (via CDK and Amplify CLI) and provision into an AWS Account, then get it configured and running to submit a tour planning problem and generate an optimized solution file. You will then be able to run a React app where the user can see the list of generates routes for each vehicle in the fleet, tap on that and get that vehicles routes for the day.

## Building solution and Infrastructure provisioning

Once you have identified or created an AWS Account, installed all the pre-requisite installations and cloned the Github repository, you are ready to begin!

### Getting Solution Dependencies
1. *Get the Node dependencies for the AWS Lambda function.*
```
cd lib/lambda/calculate-route
npm install
```

2. *Get the Node dependencies for the sample frontend application.*
```
cd frontend/here-driver-app
npm install
```

3. *Complete the "Getting Started" steps in frontend/here-driver-app/README.md*



### AWS CDK Setup
From the root of the repository, run the following command to generate the required AWS infrastructure:

If you have never used CDK in your AWS Account, you must first [bootstrap](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html) the solution, which creates some Amazon S3 buckets and metadata to support CDK operations.
```bash
cdk bootstrap
```
Then you can deploy the infrastructure for this solution using command,

```typescript
cdk deploy
```

This will provision much of the infrastructure required for this solution such as Amazon DynamoDB, API Gateway, AWS Lambda Functions and S3 buckets. It will also output an unique API Gateway Endpoint which needs to be updated in the driver app. See [README.md](frontend/here-driver-app/README.md) for additional details.


Next, Amplify will help us provision the remaining resources in order to complete the solutions

Go to the folder for the frontend:

```typescript
cd frontend/here-driver-app/
```


Run the Amplify command to create the remaining resources, AWS AppSync, Amazon Cognito, API Gateway and Amazon Location Service.

```typescript
amplify init
amplify push
amplify publish
```

You should now have all the infrastructure created to run the solution!


## Building blocks

The solution uses 4 Lambda functions, that can be found under /lib/Lambda.

1. solve-problem
    - This lambda is responsible for invoking the HERE tour planning api to solve the problem uploaded to the S3 bucket 
    - Once solved, it puts the solution json in the provisioned S3 destination bucket. 
    - It also all metadata information about the problem / solution as an entry in a DynamoDB Table. 

2. get-routes
    - This lambda reads the metadata information in DynamoDB Table to get information about Solved routing problems
    - It get the relevant objects from S3 buckets and returns a consolidated list of solved problems as a JSON document

3. get-route-detail
    - This Lambda takes as input the vehicle ID and solved problem ID in DynamoDB Table 
    - It returns a JSON document containing all the route information such as DeparturePosition, WaypointPositions and DestinationPosition

4. calculate-route
    - This Lambda uses an Amazon Location Service, Route Calculator, to create a route between two geographic locations.
    - The calculated route includes the geometry for drawing the route on a map, plus the overall time and distance of the route.

## Sample Problem and Solution Files

The 'data' folder contains sample files representing typical Problems and the corresponding Solution files returned by invoking HERE Tour Planning API. For more examples of routing Problems please see this [link](https://developer.here.com/documentation/tour-planning/3.13.0/dev_guide/topics/concepts/problem.html).

## User journey

1. Create problem and generate solution with tour planning

You can either sign up for the HERE Developer program in order to receive an API Key to test this solution live or use the problem and pre-solved solution that is provided in the /data folder of the repository.
1.	Open the S3 bucket that was created. You can find this construct in the lib/aws-here-optimize-fleet-utilization-stack.ts

```typescript
const tourPlanningProblemBucket = new s3.Bucket(this, 'AWS-HERE-FleetPlanning-Problem'
```


2.	Upload a problem file (which is just JSON) to the bucket folder

A Lambda function will get triggered once we add a new file to the S3 bucket which will then perform a synchronous call to the HERE Tour Planning API and generate a Solution JSON file. That file gets saved to S3 which then triggers another Lambda function to save that file to a DynamoDB table. The drivers then can use the example React web application in order to view the list of vehicles and the routes.

..thus the infrastructure is provisioned.

## Frontend

Let’s run the React front end app in order to see the results. The app and the GraphQL API is secured with Amazon Cognito.

To run the solution, go to the frontend
```typescript
cd frontend/here-driver-app/
```
 and issue command:

```typescript
npm start
```

A local web server will run on http://localhost:3000


In order to use the system, the user must authenticate. For this, we are using Amazon Cognito, which allows us to sign in or create a new user.

Don’t worry, you don’t need to code any of this. The functionality is provided by the Amazon Cognito Hosted UI and includes common functions such as Sign In, Sign Up, Email Verification and Forgot Password.

After authenticating, you will see the Home screen which includes the list of available vehicle and routes that were generated.


Tap on a vehicle and you will be taken to the detail of the route which is generated using Amazon Location Services



## Clean up the resources

To avoid incurring future charges, delete all resources created:

Let’s clear out Amplify resources first

```typescript
amplify pull
amplify delete
```

The below CDK command will delete the CloudFormation Stack that was used to provision all the resources

```typescript
cdk destroy
```

Note: You can leave the AWS CLI, Amplify CLI and CDK CLI installed on your computer as they occur no charges and are useful for future development

## Learning Resources 
While this is an excellent learning resource for the CDK, there are other resources that can be referenced to assist with your learning/development process.

### Official Resources
- [Developer Guide](https://docs.aws.amazon.com/cdk/latest/guide/home.html)
- [API Reference](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-construct-library.html)
- [CDK Workshop](https://cdkworkshop.com/)
- [CDK Repository](https://github.com/aws/aws-cdk)

## Conclusion

You have now learned how to use CDK, Amplify CLI and the HERE Tour Planning API in order to generate vehicle routes and deliver that route to each vehicle driver. In our next blog, we will explore enhancing this even further, including turn by turn directions and tracking assets on those vehicles as they get delivered.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
