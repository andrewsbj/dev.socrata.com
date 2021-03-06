---
layout: with-sidebar
sidebar: documentation
title: Authentication
---

Authentication is only necessary when accessing datasets that have been marked as _private_ or when making write requests (`PUT`, `POST`, and `DELETE`). For reading datasets that have not been marked as private, simply use an [application token](/docs/app-tokens.html).

There are two methods available for authentication: OAuth 2 and HTTP Basic Authentication. OAuth 2 is preferred; HTTP Basic authentication should only be used by legacy systems.

## OAuth 2

### Workflow

We support a subset of [OAuth 2](http://oauth.net/2/mechanism) -- the server-based flow with a callback URL -- which we believe is more secure than the other flows in the specification. This OAuth flow is used by several other popular API services on the web. We have made the authentication flow similar to [Google AuthSub](http://code.google.com/apis/gdata/docs/auth/authsub.html). 

To authenticate with OAuth 2, you will first need to [register your application](http://opendata.socrata.com/profile/app_tokens), which will create an app token and a secret token. When registering your application, you must preregister your server by filling out the `Callback Prefix` field), so that we can be sure that access through your application is secure even if both your tokens are stolen. The `Callback Prefix` is the beginning of the URL that you will use as your redirect URI. Generally, you'll want to provide as much of your callback URL as you can. For example, if your authentication callback is `http://my-website.com/socrata-app/auth/callback`, you might want to specify `http://my-website.com/socrata-app` as your callback URL. 

Once you have an application and a secret token, you'll be able to authenticate with the SODA OAuth 2 endpoint. You'll first need to redirect the user to the Socrata-powered site you wish to access so that they may log in and approve your application. For example:

    https://sandbox.socrata.com/oauth/authorize?client_id=YOUR_AUTH_TOKEN&response_type=code &redirect_uri=YOUR_REDIRECT_URI

Note that the `redirect_uri` here must be an absolute, secure (`https:`) URI which starts with the `Callback Prefix` you specified when you registered your application. If any of these cases fail, the user will be shown an error indicating as much. 

Should the user authorize your application, they will be redirected back to the your `redirect_uri`. For example, if I provide `https://my-website.com/socrata-app/auth/callback` as my `redirect_uri`, the user will be redirected to this URL:

    https://my-website.com/socrata-app/auth/callback?code=CODE

where CODE is an authorization code that you will use later.

If your `redirect_uri` contain a querystring, it will be preserved, and the `code` parameter will be added onto the end of it. Likewise, if you provide the optional `state` parameter in the original redirect to `/authenticate`, it will be preserved and sent back to you.

Now that the user has authorized your application, the next step is to retrieve an `access_token` so that you can perform operations on their behalf. You can do this by making the following `POST` request from your server:

    https://sandbox.socrata.com/oauth/access_token
    ?client_id=YOUR_AUTH_TOKEN
    &client_secret=YOUR_SECRET_TOKEN
    &grant_type=authorization_code
    &redirect_uri=YOUR_REDIRECT_URI
    &code=CODE

where YOUR_AUTH_TOKEN and YOUR_SECRET_TOKEN are the tokens you received when registering your app, YOUR_REDIRECT_URI is the same value as what you used previously, and CODE is the value of the `code` query parameter of the URL that the user was redirected to.

You'll receive the following response:

    { access_token: ACCESS_TOKEN }

Use this `access_token` in your requests when you have to do work on behalf of the now-authenticated user, as described below in the **Using an OAuth 2 Access Token** section.

We have a [sample app](https://github.com/socrata/oauth_sample_app_ruby) available on GitHub that illustrates how to do all of the above with the Ruby `OAuth2` gem.

### Using an OAuth 2 Access Token

Once you have obtained an `access_token`, you should include it on requests which need to happen on behalf of the user. The token must be included in the `Authorization` HTTP Header field as follows:

    Authorization: OAuth YOUR_ACCESS_TOKEN

**Note:** All authenticated requests *must* be performed over a secure connection (`https`). Any attempt to use an `access_token` over a non-secure connection will result in immediate revocation of the token.

### Who am I?

One quirk of authenticating via OAuth 2 is that the entire process happens without the 3rd party application (that's you!) having any knowledge of who, exactly, the user is that just authorized the application. To remedy this, we have set up an endpoint that simply returns the information of the current user. To return the data in JSON:

    https://sandbox.socrata.com/api/users/current.json

To return the data in XML:

    https://sandbox.socrata.com/users/current.xml

## Authenticating using HTTP Basic Authentication

Requests can be authenticated by passing in HTTP Basic Authentication headers. We only support this method for legacy purposes, and encourage all our API users to use the OAuth 2 workflow to authenticate their users. We may deprecate this authentication method in the future.

All HTTP-basic-authenticated requests *must* be performed over a secure (`https`) connection, and *must* include an application token, which is obtained when you [register your application](http://opendata.socrata.com/profile/app_tokens). Authenticated requests made over an insecure connection or without an application token will be denied.

Here is a sample using curl on how to use HTTP Basic Authentication:

    > curl --header "X-App-Token: b6bThWI6I3dJfulFek8JRizjo" --user LOGIN:PASS https://sandbox.socrata.com/resource/earthquakes.json
