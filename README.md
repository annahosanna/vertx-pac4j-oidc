# vertx-pac4j-oidc
### Purpose
* An AWS ALB OIDC headless test client for idp integration
### Attribution
* This is based on vertx-pac4j-demo
### AWS ALB OIDC Configuration Notes
* [Okta blog entry describing workflow](https://developer.okta.com/blog/2018/01/11/sso-auth-vertx)
* [AWS Demo Page](https://exampleloadbalancer.com/auth_demo.html)
(Hopefully I transcribed this correctly)
```
User sends HTTPS request to ALB
ALB checks for session cookie and redirects (302) to idp if cookie is missing
User authenticates to idp
idp redirects user (302) to alb with CODE
alb authenticates with the CODE to the idp
alb receives JWT (ID Token, Access Token) from idp
alb uses access token to get userinfo from idp
<alb caches user info?>
alb redirects to original uri with AWSELBAuthSessionCookie set (or returns a 500 if there was a problem)
User requests new uri (AWSELBAuthSessionCookie already set)
ALB validates cookie and forwards user info to backend in X-AMZN-OIDC-* headers
Backend sends response back via ALB (normal)
```

* [AWS ALB Auth Notes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html)
### The workflow per AWS mixed with my notes
* AWS: Callback is `https://<alb_fqdn>/oauth2/idpresponse`
* AWS: OAuth2 Grant type must be `Authorization code grant` (not implicit)
* AWS: ALB displays 401 Unathorized when the issuer, token or authorization endpoint is not correct or the client id/secret is not correct
* AWS: You must use S256 (might also be ES256)
* Me: Code, State, Scope, Extra Params, and redirect apprear to be passed as a part of the 302/GET
* Me: The ALB gives up trying to get a token after 2 seconds and returns a 500
* Me: The issuer uri is matched against the iss field in the token
* Me: The exp field in the token must not be more then two minutes difference 
* Me: Although the ALB may be internal when it resolves the idp hostname, it uses a public dns server, and thus the idp server must be accessable via that ip.
* Me: The ALB uses https, but does not validate the root ca (in case you are using your own internal ca)

* AWS: Customer supplied paramters
 1. Issuer (uri starting with https and does not have a trailing /)
 2. AuthorizationEndPoint (uri starting with https and does not have a trailing /)
 3. TokenEndPoint
 4. UserInfoEndpoint
 5. ClientId
 6. ClientSecret
* Optional
 1. SessionTimeout (604800)
 2. SessionCookieName (AWSELBAuthSessionCookie) User will be be authenticated if cookie does not exist.
 3. Scope (must at least be `openid`, but other scopes can be added to get more user information)
 4. AuthentiationRequestExtraParams (You can find these in the OpenID standard specification)
 5. OnUnauthenticatedRequest (deny)
* Me: OpenID has many endpoints, including encrytion, and autodiscovery (issuer_uri/.well-know/openid-configuration) but AWS does not use them.
### Test should mirror ALB requirements and behavior
* Enter only the same endpoints as used by AWS setup
* Rely on idp to authenticate user
* For any user if authentication is successful display a success message
* Keep track of a session token for session expiration or if a user needs to reauthenticate
* enable user to proceed to originally requested url