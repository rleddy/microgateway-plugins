# extauth plugin

## Summary
The purpose of the extauth plugin is to allow customers to BYO-JWT or "bring your own JWT" to Edge Microgateway (EM).  
Although Apigee provides a public certificate and can generate JWTs for you, we also want to allow customers that already have and IDP, which they use to manage developer identities, to be able to send those JWTs to the Edge Microgateway.  In this case the Microgateway should validate your IDPs JWT using the public JWK endpoint, extract the client ID from the JWT and validate that client ID in the oauth plugin.

## JWT & JWK
A JWT is a JSON Web Token that is encoded, URL-safe, JSON object that contains claims about a subject and it can be used for authentication and authorization.  A JWK (JSON Web Key) is a JSON object that contains a set of keys that can be used to validate a JWT.

View the following links to learn more about [JWTs](https://tools.ietf.org/html/rfc7519) and [JWKs](https://tools.ietf.org/html/rfc7517).


## When do I use this plugin?
You should answer Yes to all the questions below.

1. Do you have your own IDP (Okta, Ping, etc. )?
2. Do you want Edge Microgateway to validate your IDP's JWTs?

If you answered yes to the questions above, then you need to use this plugin.

## Process Summary

1. Your IDP will generate a JWT via a /token request.
2. You include that JWT on the request to Edge Microgateway.
3. The EM `extauth` plugin will validate the token using the public JWK endpoint.
4. EM will extract the client_id from the JWT.
5. EM will pass the client Id in the x-api-key header.
6. EM `oauth` plugin will validate the client Id
7. If the client Id is valid this it will forward the request to the backend.

## Prerequisites
Please complete the following tasks before you use this plugin.  

1. Create a product, developer and developer app in Apigee Edge. Read our [documentation](https://docs.apigee.com/api-platform/microgateway/2.5.x/setting-and-configuring-edge-microgateway#part2createentitiesonapigeeedge) for details.   

2. Import the client Id into Apigee Edge with the following requests. The secret is optional.  Then associate the Client Id to the Apigee product.
  * [Create the client Id](https://apidocs.apigee.com/management/apis/post/organizations/%7Borg_name%7D/developers/%7Bdeveloper_email_or_id%7D/apps/%7Bapp_name%7D/keys/create)
  * This will associate the Client Id to the Apigee product. [Add API Product to client Id](https://apidocs.apigee.com/management/apis/post/organizations/%7Borg_name%7D/developers/%7Bdeveloper_email_or_id%7D/apps/%7Bapp_name%7D/keys/%7Bconsumer_key%7D)

## Plugin configuration properties
The following properties can be set in the `extauth` stanza in the Microgateway configuration file.

```yaml
extauth:
  publickey_url: "URL to your JWK endpoint"
  client_id: "the property name in the JWT that contains the client Id" # defaults to client_id
  iss: "This should be the same issuer that is included int the JWT."
  exp: true # can be true or false, but defaults to true; use true to check for the expiry time and send an error if the token is expired.
  sendErr: true # can be either true or false, but defaults to true; set this to false if you want the extauth plugin to send an error if the JWT is invalid.
  keepAuthHeader: false # can be true or false; default is false; set this to true if you want to pass the Authorization header to the backend.
```

## Enable the plugin
In the EM configuration file (`org-env-config.yaml`) make sure that your plugin sequence is as shown below.

```yaml
plugins:
    sequence:
      - extauth
      - oauth
      # other plugins can be listed here
```

## Configure the plugin
In the same configuration file you also need to configure the `extauth` plugin in the EM configuration file.  

**Notice that I set the `client_id` property to `sub`, because Okta JWTs include the client Id in the `sub` property.**

```yaml
extauth:
  publickey_url: "https://youridp.com/oauth2/default/v1/keys"
  client_id: "sub"
  iss: "https://youridp.com/oauth2/default"
```


## Okta example
This is an example of how I used this plugin with JWTs generated by Okta.  

```yaml
...

plugins:
    sequence:
      - extauth
      - oauth

...

extauth:
  publickey_url: "https://yourinstance.oktapreview.com/oauth2/default/v1/keys"
  client_id: "sub"
  iss: "https://yourinstance.oktapreview.com/oauth2/default"
```


### Setup a Okta Authorization Server
[Documentation](https://developer.okta.com/authentication-guide/implementing-authentication/set-up-authz-server)

### Okta Scopes
[Okta Scopes](https://developer.okta.com/blog/2017/07/25/oidc-primer-part-1#whats-a-scope)
[Create custom Okta scope](https://developer.okta.com/authentication-guide/implementing-authentication/set-up-authz-server#create-scopes-optional)

### Curl to generate an Okta JWT
```bash
curl --request POST \
  --url https://yourokta.oktapreview.com/oauth2/default/v1/token \
  --header 'accept: application/json' \
  -u 'clientId:secret' \
  --header 'accept: application/json' \
  --header 'cache-control: no-cache' \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data 'grant_type=client_credentials&scope=customScope'
  ```