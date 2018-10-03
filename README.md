Documaster IDP
--------------

## Overview

Documaster IDP implements (partially) the [OAuth2](https://tools.ietf.org/html/rfc6749) and [OpenID Connect](http://openid.net/specs/openid-connect-core-1_0.html) protocols and supports the following **authorization flows**:

- Authorization code flow
- Resource owner password credentials flow

More information about OAuth2 can be found [here](https://oauth.net/2/).

### Authorization flows overview

#### Authorization code flow

This is the recommended flow for web and mobile applications. The flow starts by opening a browser and redirecting the user to the following IDP URL where the user is asked to provide credentials (usually username and password):

``` text
https://idpserver:{port}/oauth2/auth?response_type=code&client_id=client&redirect_uri=https://applicationurl.com&state=mystate&scope=openid'
```

In case of a successful login, the IDP redirects the user to the URL provided in the *redirect_uri* parameter (an authorization code is appended to the query string of the *redirect_uri*). Next, the application sends the authorization code and the same *redirect_uri* to the IDP to obtain an access token. This is done via a POST request:

``` text
curl -k -u clientId:clientSecret https://idpserver:{port}/oauth2/token-d 'grant_type=authorization_code&code=AUTHORIZATION_CODE&redirect_uri=https://applicationurl.com'
```

The response body contains a JSON object with the following format:

``` text
{
  "access_token":"MTQ0NjJkZmQ5OTM2NDE1ZTZjNGZmZjI3",
  "token_type":"bearer",
  "expires_in":3600,
  "refresh_token":"IwOGYzYTlmM2YxOTQ5MGE3YmNmMDFkNTVk",
  "scope":"create",
  "state":"12345678"
}
```

#### Resource owner password credentials flow

This flow is intended to be used by trusted applications not capable of opening a browser (typically back-end applications). To retrieve an access token, the application executes a POST request providing user and client credentials:

``` text
curl -k -u clientId:clientSecret https://idpserver:{port}/oauth2/token -d 'grant_type=password&username=USERNAME&password=PASSWORD&scope=openid'
```

The IDP returns an access token like the one described in the Authorization code flow example.

### Using access tokens

#### Obtaining user claims

The IDP allows applications to obtain claims for a particular user. To do this, the application needs to perform the following POST request:

``` text
curl -k -H "Authorization: Bearer ACCESS_TOKEN" https://idpserver:{port}/oauth2/userinfo
```

The response body is a JSON object with the following format:

``` text
{
    "preferred_username": "john",
    "email": "userX@mail.com",
    "name": "John Smith",
    "https:\/\/documaster.com\/idp\/groups": ["group1","group2"],
    "sub": "6e8c827a-4466-11e8-842f-0ed5f89f718b"
}
```

#### Refreshing access tokens

An application holding an access token is responsible for refreshing (if needed) the token once the token expires. This is done without asking the user for credentials again - the access token response from the IDP contains a refresh token that can be used to obtain a new access token:

``` text
curl -u clientId:clientSecret https://idpserver:{port}/oauth2/token -d 'grant_type=refresh_token&refresh_token=REFRESH_TOKEN'
```

### Calling Documaster web services

To call any access-controlled Documaster web service, you need to include a valid access token in the request (in the Authorization header), e.g.:

``` text
POST https://{documaster-rms-server}:{port}/rms/api/public/noark5/download?id=DOCUMENT_ID HTTP/1.1
Authorization: Bearer ACCESS_TOKEN
```

Note that you will typically call Documaster web services using the [C# client](https://github.com/documaster/ws-client-n5-csharp-samples), which facilitates the handling of tokens.

## Documaster IDP web services

### Supported protocols

The web services support the following protocols:

- HTTPS

### HTTP status codes

| Status code               | Definition                                                                                                                         | Example                                                         |
|:--------------------------|:-----------------------------------------------------------------------------------------------------------------------------------|:----------------------------------------------------------------|
| 200 OK                    | The request was successful.                                                                                                        | HTTP/1.1 200 OK                                                 |
| 400 Bad Request           | The syntax of the request was invalid. This might be an invalid content type, invalid request method or multiple tokens submitted. | HTTP/1.1 400 Bad Request                                        |
| 401 Unauthorized          | The request was denied due to a malformed, expired or missing access token.                                                        | <p>HTTP/1.1 401 Unauthorized</p><p>WWW-Authenticate: Bearer</p> |
| 403 Forbidden             | The request was denied due to the access token having insufficient privileges.                                                     | HTTP/1.1 403 Forbidden                                          |
| 404 Not Found             | The requested resource does not exist or is not accessible.                                                                        | HTTP/1.1 404 Not Found                                          |
| 500 Internal Server Error | An internal server error has occurred.                                                                                             | HTTP/1.1 500 Internal Server Error                              |
| 503 Service Unavailable   | The service is temporarily unavailable.                                                                                            | HTTP/1.1 503 Service Unavailable                                |

### Error responses

In case the IDP returns HTTP status code 400 or 401, the response body will contain a JSON-formatted object with details about the error:

``` text
{
    "error": "invalid_token",
    "error_description": "The access token provided is invalid"
}
```

### Endpoints

Address:
```
https://{idpserver}:{port}/oauth2/
```

### auth

This endpoint is used by web applications to authenticate users (authentication is performed by the IDP).

**Request**

``` text
GET /auth?response_type=code&client_id=CLIENT_ID&redirect_uri=REDIRECT_URI HTTP/1.1
```

**Details**

- **CLIENT_ID**
  - client ID
- **REDIRECT_URI**
  - URI where the user should be redirected after a successful login
  - *scope* and *state* will be appended as query parameters

### token

Get an access token using either the *authorization code flow* or the *resource owner password credentials flow*.

#### Authorization code flow

Get an access token by providing an authorization code.

**Request**

``` text
POST /token HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Authorization: Basic CLIENT_ID:CLIENT_SECRET
Body: 'grant_type=authorization_code&code=AUTHORIZATION_CODE&redirect_uri=REDIRECT_URI'
```

**Details**

- **AUTHORIZATION_CODE**
  - valid authorization code retrieved after a successful user login
- **REDIRECT_URI**
  - must match the redirect URI specified in the authorization code request
- **CLIENT_ID**
  - client ID
- **CLIENT_SECRET**
  - client secret

Note that instead of providing **CLIENT_ID** and **CLIENT_SECRET** in the *Authorization* header, it is also possible to append the values in the request body, e.g.:
```
...&client_id=CLIENT_ID&client_secret=CLIENT_SECRET
```

**Response**

``` text
Content-Type: application/json

{
  "access_token":"MTQ0NjJkZmQ5OTM2NDE1ZTZjNGZmZjI3",
  "token_type":"bearer",
  "expires_in":3600,
  "refresh_token":"IwOGYzYTlmM2YxOTQ5MGE3YmNmMDFkNTVk",
  "scope":"create",
  "state":"12345678"
}
```

#### Resource owner password credentials flow

Get an access token by providing user and client credentials.

**Request**

``` text
POST /token HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Authorization: Basic CLIENT_ID:CLIENT_SECRET
Body: 'grant_type=password&username=USERNAME&password=PASSWORD&scope=openid'
```

**Details**

- **CLIENT_ID**
  - client ID
- **CLIENT_SECRET**
  -  client secret
- **USERNAME**
  - user name
- **PASSWORD**
  - password matching the *USERNAME*

Note that instead of providing **CLIENT_ID** and **CLIENT_SECRET** in the *Authorization* header, it is also possible to append the values in the request body, e.g.:
```
...&client_id=CLIENT_ID&client_secret=CLIENT_SECRET
```

**Response**

``` text
Content-Type: application/json

{
  "access_token":"MTQ0NjJkZmQ5OTM2NDE1ZTZjNGZmZjI3",
  "token_type":"bearer",
  "expires_in":3600,
  "refresh_token":"IwOGYzYTlmM2YxOTQ5MGE3YmNmMDFkNTVk",
  "scope":"create",
  "state":"12345678"
}
```

### userinfo

Get user information.

**Request**

``` text
POST /userinfo HTTP/1.1
Authorization: Bearer ACCESS_TOKEN
```

**Details**

- **ACCESS_TOKEN**
  - valid access token

**Response**

``` text
Content-Type: application/json

{
    "preferred_username": "john",
    "email": "userX@mail.com",
    "name": "John Smith",
    "https:\/\/documaster.com\/idp\/groups": ["claim1","claim2"],
    "sub": "6e8c827a-4466-11e8-842f-0ed5f89f718b"
}
```
