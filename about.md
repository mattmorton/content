---
layout: default
title: About
permalink: /about/
---

I am a software engineer, specialising in Javascript and it's ecosystem, based in Sydney Australia. 

### Projects

***

#### AWS Lambda OAuth Proxy

An OAuth proxy that allows client applications to authenticate against any defined provider, without exposing OAuth configuration settings to the browser.

- OAuth code exchange handled by single AWS Lambda function and AWS API Gateway routes.
- Any number of providers supported by creating a JSON configuration.
- Create overrides for custom behaviour like alternative callback routes, scopes, or transport methods.

[Github](https://github.com/mattmorton/aws-lambda-oauth-proxy){:target="_blank"}

***

#### [Strava Client](https://master.d246ba5wfm1xlv.amplifyapp.com/login){:target="_blank"}

An Angular Strava client that displays a users Strava summary and recent activities.

- Google Maps integration to display activity route.
- Authentication handled by my AWS Lambda OAuth Proxy.
- 3rd part API integration (Strava API).
- Common Angular design patterns - reactive programming (rxjs), lazy loading, smart / presentational components.
- Hosted on AWS Amplify.

[Github](https://github.com/mattmorton/strava-client){:target="_blank"}

***

#### [Hackernews Client](https://master.daciud9e7j78b.amplifyapp.com){:target="_blank"}

A read-only Angular Hackernews clone.

- Nested / expandable comment section for exploratory reading.
- 3rd part API integration (Hackernews Firebase API).
- Common Angular design patterns - reactive programming (rxjs), lazy loading, smart / presentational components.
- Hosted on AWS Amplify.

[Github](https://github.com/mattmorton/hn-client){:target="_blank"}

***