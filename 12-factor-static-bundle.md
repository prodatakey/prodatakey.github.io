At Prodata Key, we agree with the tenets presented by Heroku's [Twelve-Factor App](https://12factor.net/) methodology. We hope to apply these principles to all parts of our architecture.

One part of our architecture is an Nginx Docker container which serves our web application bundle. It was difficult to determine how certain parts of Twelve Factor would apply to this. Specifically, in the ["Config" factor:](https://12factor.net/config)

> Apps sometimes store config as constants in the code. This is a violation of twelve-factor, which requires **strict separation of config from code.** [...] The twelve-factor app stores config in *environment variables*.

The question is: how can we configure this bundle, which executes on the user's device, to read data from the server's environment variables?

Here's how we did it:


## The `index.html` ##
To expose these environment variables to the client JS, we add the following to our index.html:

    <!-- Try this for an AngularJS app: -->
    <script>
        angular.module('app').constant('envRuntimeConfig', {
            //GLOBAL_CONFIG_TOKEN
        });
    </script>
    
    <!-- Or, for a more general purpose solution: -->
    <script>
        var ENV_RUNTIME_CONFIG = {
            //GLOBAL_CONFIG_TOKEN
        };
    </script>

The key here is that we provide our application a configuration object. Although the object is currently empty, (except for the comment,) it is valid JavaScript on its own, so it will still run correctly even when bundled and run through some live dev server.

What is the `//GLOBAL_CONFIG_TOKEN`? It's just a unique string that we are able to replace on the fly when the app is served through nginx. So how do we do that?


## The `nginx.conf` ##
Let's look at the relevant snippet from our nginx.conf:

    location / {
        try_files $uri $uri/ /index.html =404;
        sub_filter '//GLOBAL_CONFIG_TOKEN'  '${MY_ENV_CONSTANTS}';
        sub_filter_once on;
    }

There's an Nginx module called [http://nginx.org/en/docs/http/ngx_http_sub_module.html](ngx_http_sub_module) which is included by default in the Nginx Docker image. This module provides us with the ability to replace arbitrary strings with other strings. We specify the `sub_filter_once` option as a minor optimization so that it only tries to replace one instance of the string.

However, the syntax seen here to swap in `MY_ENV_CONSTANTS` isn't the actual syntax for injecting environment variables into nginx.conf. In fact, ["out-of-the-box, nginx doesn't support environment variables in most configuration blocks."](https://docs.docker.com/samples/library/nginx/#using-environment-variables-in-nginx-configuration) So how does this work? We need one more piece to make this work.


## The `Dockerfile` ##
We use [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) to prepare our application bundle for our Nginx image. Here is just the final stage of our Dockerfile:

    FROM nginx:alpine

    RUN apk add --no-cache tini
    ENTRYPOINT ["/sbin/tini", "--"]

    # ... copy in application bundle ...

    COPY nginx/default.template /etc/nginx/conf.d/default.template

    ENV MY_ENV_CONSTANTS ''

    CMD /bin/sh -c "envsubst '\$MY_ENV_CONSTANTS' < /etc/nginx/conf.d/default.template > /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"

There are a few things going on here, so I'll talk about each one.

The first thing I do is install tini and set it to be the entrypoint. I'll come back to this later.

After copying in our application bundle, I add our server's nginx.conf, although I call it `default.template` instead of `default.conf`.

Then, I declare the environment variable in our Dockerfile, `MY_ENV_CONSTANTS`, and give it a default value of an empty string. We will give this variable a value elsewhere, such as our docker-compose.yml.

The final line is the most crucial. Rather than immediately running nginx, I use `sh -c` to run multiple commands in a sequence. The second command is Nginx, but the first one is [`envsubst`](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html). This program takes in a template as input, substitutes in the specified environment variables, and outputs the result. Notice that `envsubst` is not executed at build time, but at runtime! This means we can substitute in MY_ENV_CONSTANTS in our docker-compose file later.

So, why do we need tini? Because Nginx is a child of our `sh` process, asking Docker to perform a graceful shutdown will fail. This is because the `SIGKILL` signal that Docker sends does not automatically propagate to child processes, and `sh` does not automatically provide this functionality. Enter `tini`, which is able to properly forward these signals to all its children. The creator of tini writes about this in more detail [here.](https://github.com/krallin/tini/issues/8)

## Specifying `MY_ENV_CONSTANTS` in docker-compose ##
Here's an example of a service in a docker-compose file which provides the needed environment variable:

    services:
      web:
        image: my-static-server
        environment:
          MY_ENV_CONSTANTS: >
            /* this is injected inside of the braces of our empty JS object! */
            apiRoot: "http://localhost:9090",
            debugLogsEnabled: true
        ports:
          - "80:80"

The `>` symbol in our yaml file begins a multi-line string. Please note that indentation *and newlines* are stripped from the string. Don't use `//newline-terminated comments` in there, or you will get unexpected results!

At this point, you should be able to launch an Nginx instance carrying a static bundle which is still configurable by its server's environment variables. Happy coding!
