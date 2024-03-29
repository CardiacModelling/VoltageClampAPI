# What appens inside the container

## The docker container runs a server

The artefacts `app` can run as an "application server".
This means that it handles HTTP requests on a port, but not one that we've exposed to the outside world.
It _can_ do this, but it isn't designed to be a full & secure web server, so we only do this for development.

Instead, we use NGINX as our web server.
NGINX will listen to requests from the outside world (in this case from outside our docker container) and pass them on to our app.
Benefits of adding NGINX into the mix include the option to serve static files and control who gets to access the app.

Finally, for reasons to do with performance and process management, we don't even use the application server provided by flask (started with `app.run()`), but instead start an application server with `gunicorn`.

```
Outside world --> NGINX --> Gunicorn + flask
```

We write the last bit as `gunicorn + flask` because they run in the same Python process: `gunicorn` imports our flask `app` module directly.

## The container port is mapped to a host port

This means that if you try to access that port on the host machine, you get redirected to the nginx server in the container.

```
                         --- Docker container ---
                         |                      |
Port on host machine --> |--> NGINX --> etc.    |
                         |                      |
                         ------------------------
```

This is done using the `-p` switch when running docker.

## Yet another server passes requests on to this port

Instead of exposing the port on the host machine that we've mapped to, we redirect to it using another server.
For example, if the host machine has an Apache server listening on port 80, it can pass requests to a certain address to port `4321`.

```
<Location /artefact-webapp>
   ProxyPass http://localhost:4321
   ProxyPassReverse http://localhost:4321
</Location>
```

So the whole thing looks like this:

```
Outside -->   Apache   -->  Container
 world      on port 80     on port 4321
```

## Why'd we have to go and make things so complicated

The simplest set-up would be to have the `app` run on a dedicated machine, listening to port 80, and without any other servers.
But if you want to share with other things, e.g. have websites running over port 80 too, it makes sense to have a second server.
So that's where apache (in the above example) comes in.

It should be possible for apache to talk directly to the `app` inside the container, via WSGI (and using e.g. `mod-wsgi`).
However, to be a bit more future-proof we are adding NGINX in between, so that we can also e.g. serve static files or use other NGINX features like access control, *without implementing them in our app, or having to mess with the main Apache server*.

Finally having everything (except the big apache server) in a container means we can use system packages to set everything up.
This is the easiest (only?) way to go, because server software has historically been designed under the assumption of a dedicated server.

## Security

We're running in a docker container, and there is no database, user information, or any sensitive data inside the container.
This means the only security risk is someone somehow changing what this container serves.
We will run nginx as root, but with worker processes running in `www-data`.
We will also run `gunicorn` as a passwordless user `vclamp`.
This means it can access, but not modify, files inside `app`.

