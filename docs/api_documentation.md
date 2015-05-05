# HTTP API Documentation

Chapters:
- [Basics](#basics)
- [OAuth2](#oauth2)
 - [Supported Grants](#supported-grants)
 - [Authorization Code Grant](#authorization-code-grant)
 - [Resource Owner Password Credentials Grant](#resource-owner-password-credentials-grant)
 - [Client Credentials Grant](#client-credentials-grant)
- [Scopes] (#scopes)
 - [Supported scopes](#supported-scopes)

## Basics

OSIAM uses the [OAuth 2.0](http://tools.ietf.org/html/rfc6749) standard the provide a secure API. The standard supports different ways of getting access to a secured API. They are called "grants". OSIAM support three different grants, while the concept of OSIAM is based on the grant providing the best security. It is important to understand the concept of the so called 'Authorization Code Grant'.

There are a lot of webpages out there explaining that topic in detail, the following sections are an OSIAM related summary.

OSIAM has two types of interfaces:
* The authx interface - also called technical interface, implementing the OAuth 2.0 standard: Used for authentication (authn) and authorization (authz)
* The resource interface - also called functional interface, implementing the SCIMv2 standard: Used for management of the data (identities) within OSIAM

OSIAM basically communicates with two parties:
* The client: Is an application that wants to work with the data stored in OSIAM
* The user: Is the one who is using the client.

OSIAM needs to know both parties to provide its services.
* The client has to be configured in OSIAM including a secret phrase, so OSIAM can verify that it is an authentic client is sending requests.
* The user needs to be stored in the OSIAM database as well, including an authentication parameter (e.g. a password) and the so called 'scopes'. The scopes define what actions an user can perform on the OSIAM resource interface.

In order for a client to access the resource interface of OSIAM the client needs to be authorized. This is done by the user. In the standard flow the user gets to see a [login screen](https://github.com/osiam/server/blob/master/auth-server/defaultConfiguration/auth-server/templates/web/login.html) provided by OSIAM to authenticate himself (e.g. using a password). If successful the client can make use of OSIAM's resource interface based on the scopes defined on the user's record.

So in short words: OSIAM needs to know at least about a single user and one client to provide its services. That is the reason why an OSIAM installation comes with the [default user and a default client](detailed_reference_installation.md#default-setup).

## OAuth2

### Supported Grants

An authorization grant is a credential representing the authorization to access protected resources. This credential is called "access token". Once a client has the access token it has to use the token everytime it accesses a protected resource until the token gets invalid.
More details can be found in the [OAuth 2.0 specifictaion](http://tools.ietf.org/html/rfc6749#section-1.3).

OSIAM supports three OAuth 2.0 grant types:
* [authorization code](#authorization-code-grant)
* [resource owner password credentials](#resource-owner-password-credentials-grant)
* [client credentials](#client-credentials-grant)

### Authorization Code Grant
This is the most secure grant and is recommended for every production use case.

With this grant the user needs to authenticate himself, the user has to authorize the client to access the protected resources and last but not least the registered client needs to authenticate himself.

OSIAM implements some additional features for the client authorization that are [configurable per client](#client-management):
* Authorization behaviour: OSIAM can skip the step of the user authorization by performing an internal implicit authorization or the a authorization once given by the user can be store as valid for a configurable period of time, instead of asking the user for authorization on every login.

#### How to get the access token?

##### Authorization Request
First the "authorization code" is required. This happens with the so called [authorization request](http://tools.ietf.org/html/rfc6749#section-4.1.1). In short words this is what happens:

The client that wants to get the authorization code redirects the user's browser to the authorization server providing several parameters shown in the example request below:

```http
https://<OSIAMHOST>:8443/osiam-auth-server/oauth/authorize?client_id=<CLIENT_ID>&response_type=code&redirect_uri=<CLIENT_REDIRECT_URI>&scope=<SCOPES>
```

This request an all consequent examples are based on an encrypted connection, if you do not have SSL enabled on your application server, the protocol prefix is only 'http://' and the port is likely to be '8080'.

The additional parameters in the example shortly explained:
* OSIAMHOST - is the hostname or IP address of the machine OSIAM is running on.
* CLIENT_ID - is the identifier of the client the client was registered with on OSIAM's client management interface.
* CLIENT_REDIRECT_URI - the redirect URI of the client. This URI must begin with the value of the redirect URI stored in OSIAM's database for that client, or it can be identical to it. So you can provide additional parameters for the redirect URI in the authorization request.
* SCOPES - the [scopes](#scopes) the client wants to be authorized for when using the access token.

##### Interactive Authorization
Now the user sees a page to authenticate himself (e.g. using a username and a password). After successful authentication and when the client is configured in the OSIAM database that way, the user will also be asked to authorize the client.

##### Authorization Response
The server now sends an [authorization response](http://tools.ietf.org/html/rfc6749#section-4.1.2) that includes the authorization code and redirects the user back to the client. If something goes wrong an [error response](http://tools.ietf.org/html/rfc6749#section-4.1.2.1) is send.

##### Access Token Request
With the authentication code the client can now talk directly to the server to request the access token. This is done via the [access token request](http://tools.ietf.org/html/rfc6749#section-4.1.3) using
[HTTP basic authentication](http://tools.ietf.org/html/rfc2617). An example access token request below:

```sh
curl -H "Authorization: Basic <BASE64_CLIENT_CREDENTIALS>" -X POST -d "code=<AUTH_CODE>&grant_type=authorization_code&redirect_uri=<CLIENT_PROVIDED_URI>" http://<OSIAMHOST>:8080/osiam-auth-server/oauth/token
```

The parameters (beside the OSIAMHOST are):
* BASE64_CLIENT_CREDENTIALS - required for the HTTP basic authentication, it consists of the CLIENT_ID and the client's SECRET
* AUTH_CODE - is the previously received authentication code.
* CLIENT_PROVIDED_URI - The URI of the client that must match the one stored for the client in OSIAM's database

##### Access Token Response
The [access token response](http://tools.ietf.org/html/rfc6749#section-4.1.4) includes the access token the client needs to use OSIAM's resource interface on the authorized scopes.

### Resource Owner Password Credentials Grant
This grant provides the possibility to get an access token without user interaction. But needs client and user credentials.
Authorization request/response from the authorization code grant is not required. is necessary, only the [Access Token Request](http://tools.ietf.org/html/rfc6749#section-4.3.2) and [HTTP Basic Authentication](http://tools.ietf.org/html/rfc2617) is used.

```sh
curl -H "Authorization: Basic <BASE64_CLIENT_CREDENTIALS>" -X POST -d "grant_type=password&username=<USERNAME>&password=<PASSWORD>&scope=<SCOPES>" http://<OSIAMHOST>:8080/osiam-auth-server/oauth/token
```

The parameters are similar to the access token request from the authorization code grant, but the user credentials are provided using the parameter
* USERNAME and
* PASSWORD

An example based on [OSIAM's default setup](detailed_reference_installation.md#default-setup):

```sh
curl -H "Authorization: Basic ZXhhbXBsZS1jbGllbnQ6c2VjcmV0" -X POST -d "grant_type=password&username=admin&password=koala&scope=GET POST PUT PATCH DELETE" http://localhost:8080/osiam-auth-server/oauth/token
```

### Client Credentials Grant
This grant provides a possibility to get an access token without user interaction and needs only client credentials, no authorization request is required, only the [Access Token Request](http://tools.ietf.org/html/rfc6749#section-4.4.2) and [HTTP Basic Authentication](http://tools.ietf.org/html/rfc2617) is used.

```sh
curl -H "Authorization: Basic <BASE64_CLIENT_CREDENTIALS>" -X POST -d "grant_type=client_credentials&scope=<SCOPES>"
http://<OSIAMHOST>:8080/osiam-auth-server/oauth/token
```

An example based on [OSIAM's default setup](detailed_reference_installation.md#default-setup)

```sh
curl -H "Authorization: Basic ZXhhbXBsZS1jbGllbnQ6c2VjcmV0" -X POST -d "grant_type=client_credentials&scope=GET POST PUT PATCH DELETE" http://localhost:8080/osiam-auth-server/oauth/token
```

## Scopes

Scopes are used to define access rights, see [OAuth2 Spec](http://tools.ietf.org/html/draft-ietf-oauth-v2-31#section-1.4) for further details.

### Supported scopes

OSIAM implements the different http methods as single scopes plus the scope 'email':

* GET - allows you all get calls it stands for reading,
* POST - allows you all post calls it stands mostly for creating
* PUT - allows you all put calls it stands for replacing
* PATCH - allows you all patch calls which means updating
* DELETE - allows you all delete calls which means deleting
* email - allows the access to a basic dataset of the user was authenticated during the OAuth 2.0 flow