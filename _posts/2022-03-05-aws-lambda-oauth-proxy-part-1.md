---
layout: post
title:  "Serverless Oauth - AWS Lambda + Grant"
date:   2022-03-05 08:23:13 +1100
---

## Implementing OAuth Flows

OAuth providers can support any of the grant types, depending on the use case. For web based applications, there are generally 2 flows:
- Implicit Flow: A simplified flow where an access token is returned to the client. This flow is no longer recommended, and is typically no not supported by providers.
- Authorization Code Flow: A more secure flow where an authorization code is returned to the client, which is then exchanged for an access token. 
- Proof Key for Code Exchange (PKCE): An extension of the Authorization Code Flow, which uses a code verifier and challenge instead of a client secret to authenticate.

For browser applications, PCKE is the safest authentication option, but I have found many providers do not support it yet, which means the Authorization Code Flow without PKCE is the next best option.

Provisioning and managing a server just to manage authentication can be overkill depending on your scenario - for me, I wanted a simple setup that I could use for any provider, given I provide a configuration object for the provider. My client application should be able to invoke the authorization flow, and receive a callback with the access token, without exposing any of the oauth configuration to the browser.

## Setup

1. **[Grant][grant]** is a library that handles OAuth requests to an authorization server, and can be used with popular Node.js HTTP frameworks and cloud services. It also supports many different OAuth providers, which can be setup with some JSON configuration.

2. **[AWS Lambda][aws-lambda]** is a service that lets you run code without needing a server.

3. **[AWS API Gateway][aws-api-gateway]** is a service for running REST, HTTP or Websocket APIs.

You will need an AWS account to setup this up completely.

[Here's a link](https://github.com/mattmorton/aws-lambda-oauth-proxy) to my github repository for the OAuth Code if you'd rather just clone it.

## OAuth Code

Create a project folder, initialise a node project and install the Grant library:
{% highlight shell %}
mkdir oauth-proxy
cd oauth-proxy
npm init
npm i -S grant
npm i
{% endhighlight %}

Create an index.js file to contain the main function - this is essentially unchanged from the [Grant library example for AWS][grant-aws-example]:

{% highlight javascript %}
var config = require('./config');
var grant = require('grant').aws({
  config, session: {secret: process.env.SESSION_SECRET }
})
exports.handler = async (event) => {
  var {redirect, response} = await grant(event)
  return redirect || {
    statusCode: 200,
    headers: {'content-type': 'application/json'},
    body: JSON.stringify(response)
  }
}
{% endhighlight %}

Create a config.js file to contain the grant configuration.

{% highlight javascript %}
module.exports = {
  defaults: {
    origin: process.env.HOST,
    transport: "state"
  },
  github: {
    key: process.env.GITHUB_CLIENT_ID,
    secret: process.env.GITHUB_CLIENT_SECRET
  }
}
{% endhighlight %}

This is the most basic configuration, but needs a bit of work to setup the provider. For Github,
[register a new OAuth application][github-new-oauth-application] to get a client id and secret. For now, just set the Authorization callback URL to `http://localhost:3000` - this will be updated later once we have a url from AWS API Gateway. 

All the `process.env` variables will be set in AWS.

Finally, we need to zip up the codebase for upload to AWS:

{% highlight shell %}
zip -r lambda.zip *
{% endhighlight %}

That's it for the code! Now there are a couple of components to wire together on AWS and the provider app to get this working. 

## AWS Lambda - Part 1

On the AWS console, navigate to [Lambda][aws-console-lambda], select Functions from the nav menu and select Create function. Give the function a name, and accept the default settings, including the default execution role - this will create a role to give us log access in [Cloudwatch][aws-console-cloudwatch].

Once the function is created, from the code tab, select upload from .zip file, and choose the zipfile created earlier. 

![AWS Lambda initial code screenshot](/assets/aws_lambda_init_code.png)

Alternatively, if you have the AWS CLI configured, upload the .zip from the command line using the [update-function-code command][aws-cli-update-function-code]:

{% highlight shell %}
aws lambda update-function-code --function-name <FUNCTION_NAME> --zip-file fileb://lambda.zip
{% endhighlight %}

## AWS API Gateway

Navigate to [API Gateway][aws-console-api-gateway], and select create API, then select build HTTP API. 

For integrations, link the Lambda function just created. 

![AWS API Gateway Configuration Step 1](/assets/aws_api_gateway_config_step_1.png)

For routes, we only need 2 to get this setup working:
* GET /connect/github - navigate to this endpoint to start the authorization process.
* GET /connect/github/callback - the response from the authorization server.

![AWS API Gateway Configuration Step 2](/assets/aws_api_gateway_config_step_2.png)

Use the default stage, and create the API. Once created, grab the invoke URL from the details screen - we need to set this as the `process.env.HOST` variable in the Lambda.

## AWS Lambda - Part 2

Select Configuration from the lambda nav menu, then Environment variables from the side menu. Here, we need to set 4 values:
* `CLIENT_ID` - from the github OAuth app configuration.
* `CLIENT_SECRET` - from the github OAuth app configuration.
* `HOST` - the invoke URL from the API Gateway.
* `SESSION_SECRET` - any sufficiently random string.

![AWS Lambda Configuration](/assets/aws_lambda_configuration.png)

## OAuth App

[Open the github OAuth app][github-developer-settings] created earlier, and update the Authorization callback URL with the API Gateway invoke URL. For github OAuth apps, we just need to provide the host - not the full callback path.

## Testing

To check if it works, hit the invoke url in the browser: `{INVOKE_URL/connect/github`. If setup correctly, we will get presented with the github login screen:

![AWS API Gateway Configuration Step 2](/assets/github_oauth_proxy_authorization.png)

And on login, we will redirect to `{INVOKE_URL}/connect/github/callback` with an `access_token` payload value.

## Next Steps

We now have simple setup to obtain access token for github. But the solution is pretty limited at the moment - we can only authenticate against a single provider, and the flow finishes at the lambda endpoint. Because we are solving the issue of authenticating from client applications, we need to return the tokens to the client URL. Luckily, Grant provides a solution to this with the concept of overrides, which allows us to modify the default behaviour for certain scenarios. We will address this in [Part 2](aws-lambda-oauth-proxy-part-2).


[github-new-oauth-application]: https://github.com/settings/applications/new
[github-developer-settings]: https://github.com/settings/developers
[grant]: https://github.com/simov/grant
[aws-lambda]: https://docs.aws.amazon.com/lambda/latest/dg/welcome.html
[aws-cli-update-function-code]: https://docs.aws.amazon.com/cli/latest/reference/lambda/update-function-code.html
[aws-api-gateway]: https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html
[grant-aws-example]: https://github.com/simov/grant-aws
[aws-console-lambda]: https://console.aws.amazon.com/lambda
[aws-console-api-gateway]: https://console.aws.amazon.com/apigateway
[aws-console-cloudwatch]: https://console.aws.amazon.com/cloudwatch