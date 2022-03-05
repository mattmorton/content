---
layout: post
title:  "Serverless Oauth - AWS Lambda + Grant: Part 2"
date:   2022-03-05 08:23:13 +1100
---

## Part 1 Recap

In [Part 1](aws-lambda-oauth-proxy-part-1), we created a small OAuth application using the Grant library, and run it on AWS Lambda. The Lambda was integrated with 2 endpoints on API Gateway, to handle initialising the OAuth flow, and receiving the callback with the access token.

We want to customise the behaviour so that the callback URL can be a client URL rather than the API Gateway endpoint. The current implementation will receive the tokens to the callback, but the client invoking the OAuth flow can't receive the tokens. Fortunately, Grant provides some configuration options to enable this.

## Customising Providers

In your OAuth code, edit the config.js file:

{% highlight javascript %}
module.exports = {
  defaults: {
    origin: process.env.HOST,
    transport: "state"
  },
  github: {
    key: process.env.GITHUB_CLIENT_ID,
    secret: process.env.GITHUB_CLIENT_SECRET,
    overrides: {
      redirect: {
        transport: "querystring",
        response: ["tokens"],
        dynamic: ["callback"]
      }
    }
  }
}
{% endhighlight %}

In the updated config, we added an overrides object to the github provider. Each override key represents a new URL path, which when called will complete the OAuth flow with its custom config. In our configuration:
* **Transport** returns the payload as a querystring rather than response state - this way we can parse the URL to get the tokens
* **Response** returns just the tokens, rather than the full response - this limits the size of the query string
* **Dynamic** callback to a custom URL, that we provide as a query param in the authorization request.

We now have 2 potential authorization endpoints:
* GET /connect/github - the original implementation
* GET /connect/github/redirect?callback={callback} - the new implementation, which will callback to a client running on localhost. The callback param is dynamic, so any valid URL can be provided.

## AWS API Gateway

Navigate to [API Gateway][aws-console-api-gateway], select our API, and select Routes from the nav menu. Create another route, and make sure the integration is attached:
* GET /connect/github/redirect

![AWS API Gateway Redirect Routes](/assets/aws_api_gateway_redirect_route.png)

## Testing

To check if it works, hit the new invoke url in the browser, and pass it a callback url: `{INVOKE_URL/connect/github/redirect?callback=http://localhost:3000`. We don't have anything running on localhost:3000, but the browser will attempt to route to it anyway - if setup correctly, we will have the `access_token` in the query params.

## Next Steps

We now have simple setup to obtain access token for github, and the ability to redirect to any callback URL we provide. Next, we will look at configuring another provider, and setting up API Gateway to handle any provider URL we call in Part 3 (coming soon).

[github-new-oauth-application]: https://github.com/settings/applications/new
[grant]: https://github.com/simov/grant
[aws-lambda]: https://docs.aws.amazon.com/lambda/latest/dg/welcome.html
[aws-cli-update-function-code]: https://docs.aws.amazon.com/cli/latest/reference/lambda/update-function-code.html
[aws-api-gateway]: https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html
[grant-aws-example]: https://github.com/simov/grant-aws
[aws-console-lambda]: https://console.aws.amazon.com/lambda
[aws-console-api-gateway]: https://console.aws.amazon.com/apigateway
[aws-console-cloudwatch]: https://console.aws.amazon.com/cloudwatch