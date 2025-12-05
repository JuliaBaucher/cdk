# Chatbot CDK – Infrastructure README (Copy-paste Friendly TXT Format)

This document explains how the Chatbot CDK infrastructure was created using AWS CDK (Python), Lambda (Node.js), Amazon Bedrock, and API Gateway HTTP API.  
Everything is written in plain text so formatting is preserved when copying into a .txt file.

----------------------------------------------------------------------
1. PREREQUISITES
----------------------------------------------------------------------

You need:

• AWS account  
• AWS CLI installed and configured in Ubuntu WSL  
• Region: eu-central-1 (Frankfurt)  
• Node.js + npm  
• AWS CDK CLI  
• Python 3 (for CDK)  

Verify AWS CLI:  
aws sts get-caller-identity  

Verify region:  
aws configure get region  


----------------------------------------------------------------------
2. PROJECT STRUCTURE
----------------------------------------------------------------------

The project uses this directory layout:

chatbot-cdk/
    resources/
        JuliaBaucher_AskMe-lambda.zip   (Node.js Lambda ZIP)
    infra/
        app.py                          (CDK entry)
        cdk.json
        requirements.txt
        .venv/                          (virtualenv)
        infra/
            __init__.py
            infra_stack.py              (Our CDK infrastructure stack)


The Lambda ZIP already contains valid Node.js code.


----------------------------------------------------------------------
3. INITIALIZE CDK (Created Earlier)
----------------------------------------------------------------------

The CDK application was created using:

cdk init app --language python

This generated:
- app.py  
- cdk.json  
- .venv virtual environment  
- infra/ python package  
- requirements.txt  


----------------------------------------------------------------------
4. PYTHON ENVIRONMENT AND DEPENDENCIES
----------------------------------------------------------------------

Activate virtual environment:

source .venv/bin/activate

Install dependencies:

python -m pip install -r requirements.txt  
pip install aws-cdk-lib constructs  


----------------------------------------------------------------------
5. CDK APP ENTRY (app.py)
----------------------------------------------------------------------

File: infra/app.py  
Purpose: define environment + load InfraStack.

CONTENT:

#!/usr/bin/env python3
import os
import aws_cdk as cdk

from infra.infra_stack import InfraStack

app = cdk.App()

env = cdk.Environment(
    account=os.getenv("CDK_DEFAULT_ACCOUNT"),
    region="eu-central-1"
)

InfraStack(
    app,
    "AskMeChatbotStack",
    env=env,
)

app.synth()


----------------------------------------------------------------------
6. INFRASTRUCTURE STACK (infra_stack.py)
----------------------------------------------------------------------

This stack:

• Creates Node.js Lambda using ZIP  
• Configures minimal Bedrock IAM permissions  
• Creates HTTP API (API Gateway v2) with CORS for GitHub Pages  
• Adds POST /chat route  
• Outputs API URL  

File: infra/infra/infra_stack.py

CONTENT:

import aws_cdk as cdk
from aws_cdk import (
    Stack,
    Duration,
    aws_lambda as _lambda,
    aws_iam as iam,
    aws_apigatewayv2 as apigwv2,
    aws_apigatewayv2_integrations as integrations,
)
from constructs import Construct


class InfraStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # Lambda function (Node.js)
        askme_lambda = _lambda.Function(
            self,
            "AskMeLambda",
            runtime=_lambda.Runtime.NODEJS_20_X,
            handler="index.handler",
            code=_lambda.Code.from_asset(
                "../resources/JuliaBaucher_AskMe-lambda.zip"
            ),
            timeout=Duration.seconds(30),
            environment={
                "BEDROCK_MODEL_ID": "eu.anthropic.claude-3-haiku-20240307-v1:0",
                "SYSTEM_MESSAGE":
                    "You are a helpful AI assistant named AskMe. "
                    "You respond to questions about the VSP and Vesper models. "
                    "Answer politely and concisely."
            },
        )

        # Minimal Bedrock permissions
        askme_lambda.add_to_role_policy(
            iam.PolicyStatement(
                actions=[
                    "bedrock:InvokeModel",
                    "bedrock:InvokeModelWithResponseStream",
                ],
                resources=["*"],
            )
        )

        # HTTP API + CORS
        lambda_integration = integrations.HttpLambdaIntegration(
            "AskMeChatIntegration",
            askme_lambda
        )

        http_api = apigwv2.HttpApi(
            self,
            "AskMeHttpApi",
            cors_preflight=apigwv2.CorsPreflightOptions(
                allow_headers=[
                    "Content-Type",
                    "Authorization",
                    "X-Requested-With",
                ],
                allow_methods=[
                    apigwv2.CorsHttpMethod.POST,
                    apigwv2.CorsHttpMethod.OPTIONS,
                ],
                allow_origins=[
                    "https://juliabaucher.github.io",
                ],
                allow_credentials=False,
                max_age=Duration.seconds(300),
            ),
        )

        # POST /chat route
        http_api.add_routes(
            path="/chat",
            methods=[apigwv2.HttpMethod.POST],
            integration=lambda_integration,
        )

        # Output base URL
        cdk.CfnOutput(
            self,
            "ApiBaseUrl",
            value=http_api.api_endpoint,
        )


----------------------------------------------------------------------
7. BOOTSTRAP THE CDK ENVIRONMENT
----------------------------------------------------------------------

CDK needs bootstrap resources before deploying:

Get account ID:
aws sts get-caller-identity --query Account --output text

Then bootstrap:
cdk bootstrap aws://ACCOUNT_ID/eu-central-1


----------------------------------------------------------------------
8. SYNTHESIS & DEPLOYMENT
----------------------------------------------------------------------

To build CloudFormation template:
cdk synth

To deploy:
cdk deploy

CDK will output something like:

ApiBaseUrl = https://xxxxx.execute-api.eu-central-1.amazonaws.com

Your chatbot endpoint is:
https://xxxxx.execute-api.eu-central-1.amazonaws.com/chat


----------------------------------------------------------------------
9. FRONTEND INTEGRATION (GitHub Pages)
----------------------------------------------------------------------

Set the API URL in your JavaScript frontend:

const API_URL = "https://xxxxx.execute-api.eu-central-1.amazonaws.com/chat";

fetch(API_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ message: text }),
})
.then(res => res.json())
.then(data => console.log(data.reply));


----------------------------------------------------------------------
10. MAIN ISSUES WE FACED AND HOW WE FIXED THEM
----------------------------------------------------------------------

ISSUE 1: "SSM parameter /cdk-bootstrap/... not found"
Cause: CDK environment was not bootstrapped.  
Fix: run "cdk bootstrap" with your account & region.

ISSUE 2: AWS::EarlyValidation::ResourceExistenceCheck
Cause: Explicit Lambda and API names conflicted with existing manual resources.  
Fix: removed "function_name" and "api_name" so CDK generates unique names.

ISSUE 3: Managed Policy "AmazonBedrockFullAccess" caused validation warnings
Cause: CDK validation rejected fromAwsManagedPolicyName.  
Fix: replaced with minimal inline policy:  
bedrock:InvokeModel  
bedrock:InvokeModelWithResponseStream

ISSUE 4: Wrong Lambda runtime
Cause: Lambda ZIP contained Node.js but CDK configured Python.  
Fix: switched runtime to NODEJS_20_X and handler to index.handler.

ISSUE 5: CORS causing "TypeError: Failed to fetch"
Cause: CORS must be handled by API Gateway, not Lambda.  
Fix: configured CORS in HttpApi using cors_preflight and removed CORS headers from Lambda.


----------------------------------------------------------------------
11. CREATING ANOTHER CHATBOT WITH SAME CDK
----------------------------------------------------------------------

You can create a second chatbot by:

1. Adding a new ZIP under resources/ or reusing the same ZIP with a different SYSTEM_MESSAGE.
2. Creating a second stack file (e.g., second_bot_stack.py) that defines a new Lambda and new HttpApi.
3. Adding the second stack in app.py:
   SecondBotStack(app, "SecondChatbotStack", env=env)
4. Running:
   cdk synth
   cdk deploy

CDK will output a separate endpoint for the second bot.


----------------------------------------------------------------------
12. SUMMARY
----------------------------------------------------------------------

You now have:

• A fully working CDK-managed Bedrock chatbot  
• Lambda (Node.js) + API Gateway HTTP API  
• Automatic CORS configured for GitHub Pages  
• Minimal IAM permissions for Bedrock  
• Clean, repeatable deployments  
• Ability to add more chatbots by creating more stacks  

This infrastructure is stable, scalable, and easy to extend.
