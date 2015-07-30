oauth_proxy
=================

Note (7/2015) - bit.ly has renamed their project to oauth2_proxy and it now works natively with several other providers, possibliy eliminating the need for this fork.  Check it out [here](https://github.com/bitly/oauth2_proxy)

This is a fork of bit.ly's google_auth_proxy project that works for any oauth provider (where any has actually been tested on Github and Google, YMMV)

A reverse proxy that provides authentication using an oauth server to validate 
individual accounts.

## Architecture

```
    _______       ___________________       __________
    |Nginx| ----> |   oauth_proxy   | ----> |upstream| 
    -------       -------------------       ----------
                          ||
                          \/
                  [oauth2 api]
```


## Installation

1. [Install Go](http://golang.org/doc/install)
2. `$ go get github.com/kevin1024/oauth_proxy`. This should put the binary in `$GOROOT/bin`

## OAuth Configuration

You will need to register an OAuth application with your chosen oauth provider, and configure it with Redirect URI(s) for the domain you
intend to run oauth_proxy on.

1. Visit the provider's API console
2. under "API Access", choose "Create an OAuth 2.0 Client ID"
3. Edit the application settings, and list the Redirect URI(s) where you will run your application. For example: 
`https://internalapp.yourcompany.com/oauth2/callback`
4. Make a note of the Client ID, and Client Secret and specify those values as command line arguments

## Command Line Options

```
Usage of ./gooauth_proxy:
  -authenticated-emails-file="": authenticate against emails via file (one per line)
  -client-id="": the  OAuth Client ID: ie: "123456.apps.googleusercontent.com"
  -client-secret="": the OAuth Client Secret
  -cookie-domain="": an optional cookie domain to force cookies to
  -cookie-secret="": the seed string for secure cookies
  -header-basic-auth=false: use Authorization: Basic header for authorization (see RFC 2617 section 2)
  -google-apps-domain="": authenticate against the given google apps domain
  -http-address="127.0.0.1:4180": <addr>:<port> to listen on for HTTP clients
  -pass-basic-auth=true: pass HTTP Basic Auth information to upstream
  -redirect-url="": the OAuth Redirect URL. ie: "https://internalapp.yourcompany.com/oauth2/callback"
  -upstream=[]: the http url(s) of the upstream endpoint. If multiple, routing is based on path
  -version=false: print version string
  -login-url: the OAuth Login URL
  -redemption-url: the OAuth code redemption URL
  -user-verification-command: Path to a script that takes the auth token and returns whether to allow the authorization request.
```


## Example Configuration

This example has a [Nginx](http://nginx.org/) SSL endpoint proxying to `oauth_proxy` on port `4180`. 
`oauth_proxy` then authenticates requests for an upstream application running on port `8080`. The external 
endpoint for this example would be `https://internal.yourcompany.com/`.

An example Nginx config follows. Note the use of `Strict-Transport-Security` header to pin requests to SSL 
via [HSTS](http://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security):

```
server {
    listen 443 default ssl;
    server_name internal.yourcompany.com;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/cert.key;
    add_header Strict-Transport-Security max-age=1209600;

    location / {
        proxy_pass http://127.0.0.1:4180;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_connect_timeout 1;
        proxy_send_timeout 30;
        proxy_read_timeout 30;
    }
}
```

An example commandline that works with github is:

```bash
/oauth_proxy --client-id="f4dddfabbebe5ba" --client-secret="ecb0561717bbf29956f" --upstream="http://localhost:8080/" --cookie-secret="secretsecret" --login-url="https://github.com/login/oauth/authorize" --redirect-url="http://localhost:4180/oauth2/callback/" --redemption-url="https://github.com/login/oauth/access_token --oauth-scope=user, --user-verification-command=/bin/verify_github_team.py"
```

## User Verification Command

Sometimes, getting an auth token from the oauth2 provider is not sufficient to gain access to your backend.  For example, maybe you want to make sure that the person authenticating belongs to a specific Github organization.  For this, you can create a user verification command.  You can see an [example script](contrib/verify_github_team.py) in contrib.

## Environment variables

The environment variables `client_id`, `client_secret` and `cookie_secret` can be used in place of the corresponding command-line arguments.

## Endpoint Documentation

oauth_proxy proxy responds directly to the following endpoints. All other endpoints will be authenticated.

* /oauth2/sign_in - the login page, which also doubles as a sign out page (it clears cookies)
* /oauth2/start - a URL that will redirect to start the oauth cycle
* /oauth2/callback - the URL used at the end of the oauth cycle
