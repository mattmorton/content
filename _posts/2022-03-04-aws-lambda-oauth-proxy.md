---
layout: post
title:  "Serverless Oauth - AWS Lambda + Grant"
date:   2022-03-04 08:23:13 +1100
categories: aws javascript lambda
---

## OAuth Code
Create a project folder, initialise a node project and install the Grant library:
{% highlight shell %}
mkdir oauth-proxy
cd oauth-proxy
npm init
npm i -S grant
npm i
{% endhighlight %}

Create an index.js file to contain the function - this is essentially unchanged from the library example:

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

Create a config.js file to contain the grant configuration. I've made this a js file exporting JSON:

{% highlight javascript %}
module.exports = {
  defaults: {
    origin: process.env.HOST,
    prefix: "/connect",
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

## Configuration
There are a couple of components to wire together on AWS and the provider app to get this working. 

### AWS Lambda - Part 1
On the AWS console, navigate to [Lambda][aws-console-lambda], select Functions from the nav menu and select Create function. Give the function a name, and accept the default settings, including the default execution role - this will create a role to give us log access in [Cloudwatch][aws-console-cloudwatch].

Once the function is created, from the code tab, select upload from .zip file, and choose the zipfile created earlier. 

![AWS Lambda initial code screenshot](/assets/aws_lambda_init_code.png)

### AWS API Gateway
Navigate to [API Gateway][aws-console-api-gateway], and select create API, then select build HTTP API. 

For integrations, link the Lambda function just created. 

![AWS API Gateway Configuration Step 1](/assets/aws_api_gateway_config_step_1.png)

For routes, we only need 2 to get this setup working:
* GET /connect/github - navigate to this endpoint to start the authorization process.
* GET /connect/github/callback - the response from the authorization server.

![AWS API Gateway Configuration Step 2](/assets/aws_api_gateway_config_step_2.png)

Just use the default stage, and create the API. Once created, grab the invoke URL from the details screen - we need to set this as the `process.env.HOST` variable in the lambda

### AWS Lambda - Part 2
Select Configuration from the lambda nav menu, then Environment variables from the side menu. Here, we need to set 4 values:
* `CLIENT_ID` - from the github OAuth app configuration.
* `CLIENT_SECRET` - from the github OAuth app configuration.
* `HOST` - the invoke URL from the API Gateway.
* `SESSION_SECRET` - any sufficiently random string.

### OAuth App
Open the github OAuth app created earlier, and update the Authorization callback URL with the API Gateway invoke URL. For github OAuth apps, we just need to provide the host - not the full callback path.

## Testing
To check if it works, we just need to hit the authorization url in the browser: `{INVOKE_URL}/connect/github`. If setup correctly, we should get presented with the github login screen:
![AWS API Gateway Configuration Step 2](/assets/github_oauth_proxy_authorization.png)

### Cloudwatch

[github-new-oauth-application]: https://github.com/settings/applications/new
[aws-console-lambda]: https://console.aws.amazon.com/lambda
[aws-console-api-gateway]: https://console.aws.amazon.com/apigateway
[aws-console-cloudwatch]: https://console.aws.amazon.com/cloudwatch