# Create A Private Static Site With OAuth For Free With Render

This blueprint helps you deploy a private [Quarto](https://quarto.org/) site behind OAuth, **for free**, on [Render](https://render.com/). This can be any static site and is not specific to Quarto.

Instructions and background are [here](https://github.com/hamelsmu/oauth-tutorial/blob/main/simple/README.md#render).

## Usage

1. Follow [these instructions](https://github.com/hamelsmu/oauth-tutorial/blob/main/simple/README.md#render) to set things up.
1. Update/author your content.  
2. Add/delete emails from [email_list.txt](./email_list.txt) based upon who is allowed to view your site.  The email(s) must correpond to a person's primary email on Github (users will sign in on GitHub to identify themselves).
2. Run `quarto render` to update your site, and **check-in all your content to GitHub.**

Your site will automatically render.

## How does this work?

### The `render.yaml` Blueprint

The [render.yaml](./render.yaml) file drives all the settings and saves you from having to click around in the Render UI. Here is an explanation of this file:

```yaml
services:
  - type: web   # Tells Render you wish to use their "Web Service" product
    name: oauth2-proxy-render   # Names your project so you can find it on your dashboard
    plan: free    # Tells render you want to use the free plan
    env: docker   # Tells render you wish to build and run a Docker container
    envVars:  # These set enviornment variables that are passed to the Docker container
      - key: OAUTH2_PROXY_CLIENT_ID      
        sync: false            # `sync: false` means to prompt the user for the value in the Render UI when first setting this up
      - key: OAUTH2_PROXY_CLIENT_SECRET
        sync: false            # `sync: false` means to prompt the user for the value in the Render UI when first setting this up
      - key: OAUTH2_PROXY_COOKIE_SECRET
        generateValue: true    # Generates a random string for you
      - key: OAUTH2_PROXY_HTTP_ADDRESS
        value: ":10000" # Render will automatically detect this port and send traffic to it.
```

This file is called a Blueprint, which has many more options [you can read about here](https://render.com/docs/blueprint-spec).  Render automatically detects the port listening for `http` traffic and routes traffic to it accordingly. Render handles the incoming `https` traffic upstream with its own load balancers, [as described here](https://community.render.com/t/how-ports-and-https-are-handled-for-docker-deploy/363/2).

### The Dockerfile

The [Dockerfile](./Dockerfile) copies the files for the static site, located in [_site/](./_site/), as well as the email whitelist [email_list.txt](./email_list.txt).  

The commands for this Dockerfile are explained [in this tutorial](https://github.com/hamelsmu/oauth-tutorial/tree/main/local).  One key difference is that we are [passing environment variables instead of flags](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview#environment-variables) for certain options. These environment variables are set in `render.yaml`, as discussed above.

```Dockerfile
FROM quay.io/oauth2-proxy/oauth2-proxy

COPY email_list.txt /site_config/
COPY _site /app/

ENTRYPOINT ["/bin/oauth2-proxy", \
            "--provider", "github", \
            "--upstream", "file:///app/#/", \
            "--authenticated-emails-file", "/site_config/email_list.txt", \
            "--scope=user:email", \
            "--cookie-expire=0h0m30s", \
            "--session-cookie-minimal=true", \
            "--skip-provider-button=true"]
```

_You don't have to deploy things this way.  Instead, you can instead [mount a disk](https://render.com/docs/disks) to this Docker container and [transfer](https://render.com/docs/disks#transferring-files) files to your site.  This is more efficient since only your static files will normally change, not the software running the proxy.   However, I want to keep things as simple as possible for this tutorial, so I'll leave that as an exercise to the reader (you may also have to pay a minimal amount for a disk).  If you do this, you will also want to [turn `Auto Sync` off](https://render.com/docs/infrastructure-as-code#:~:text=Turning%20Off%20Automatic%20Sync,and%20apply%20the%20displayed%20changes.)._  

## Further Reading

See [The tutorial](https://github.com/hamelsmu/oauth-tutorial).


