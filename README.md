Nginx Routing on Heroku
===================

Sometimes, we want to be able to route requests based on URL paths easily, or have a front end app that serve data from private back end services. For example, if you have several services in a Private Space that serve HTML, you might not want to expose them to the internet, and rather have a routing app that in front which will handle this. Or you might want to route your asset delivery to S3, and completely offload that processing from the local app server. Nginx is a powerful, fast and lightweight web server which can also operate as a reverse proxy. It also runs well on Heroku.

For this example, we will set up 3 apps

* ggn-nginx-router-test
* ggn-nginx-test-target
* ggn-nginx-test-blog

ggn-nginx-router-test will be running Nginx, with configs set to route requests back to the target app.
ggn-nginx-test-target will simply deliver HTML with the requestors HTML headers.
ggn-nginx-test-blog will simply be a second HTML target.

**Setup nginx-test-target**

Install PHP and Composer Locally (`brew install php && brew install composer` on a Mac) 

Set up your working directory and create a `.gitignore` file

```
☯ git init .
☯ heroku create ggn-nginx-test-target
```

```
☯ cat .gitignore 
/vendor/
```

Create a `Procfile`:

```
☯ cat procfile
web: vendor/bin/heroku-php-apache2
```

Create a `composer.json` :

```
☯ cat composer.json 
{
  "require" : {
  },
  "require-dev": {
    "heroku/heroku-buildpack-php": "*"
  }
}
```

Run `composer update`

Create an `index.php`:

```
☯ cat index.php 
<h1>This is not a Blog<br />
Requestor Headers:</h1>
<br />
<?php 

foreach (getallheaders() as $name => $value) {
    echo "$name: $value\n<br />";
}
?>
```

**Setup nginx-router-test**

Set up your working directory 

```
☯ git init .
☯ heroku create ggn-nginx-router-test
```

Add the nginx buildpack:

```
☯ heroku buildpacks:add https://github.com/heroku/heroku-buildpack-nginx
Buildpack added. Next release on ggn-nginx-router-test will use heroku-community/nginx.
Run git push heroku master to create a new release using this buildpack.
```

Add a `Procfile`:

```
☯ cat Procfile
web: bin/start-nginx-solo
```

Add a Nginx config file. We use an `.erb` file as the build pack will process it, and generate a finished config file out of it. It needs to go in `config/nginx.conf.erb`

```
☯ cat config/nginx.conf.erb 
daemon off;
# Heroku dynos have at least 4 cores
worker_processes <%= ENV['NGINX_WORKERS'] || 4 %>;

events {
  use epoll;
  accept_mutex on;
  worker_connections <%= ENV['NGINX_WORKER_CONNECTIONS'] || 1024 %>;
}

http {
  gzip on;
  gzip_comp_level 2;
  gzip_min_length 512;

  server_tokens off;
  
  log_format main '$time_iso8601 - $status $request - client IP: $http_x_forwarded_for - <%= ENV['DYNO'] %> to $upstream_addr - upstream status: $upstream_status, upstream_response_time $upstream_response_time, request_time $request_time';
  access_log /dev/stdout main;
  error_log /dev/stdout notice;
  log_not_found on;

  include mime.types;
  default_type application/octet-stream;
  sendfile on;

  # Must read the body in 5 seconds.
  client_body_timeout <%= ENV['NGINX_CLIENT_BODY_TIMEOUT'] || 5 %>;

  upstream upstream_production {
    server ggn-nginx-test-target.herokuapp.com;
  }

  server {
    listen <%= ENV["PORT"] %>;
    server_name _;

    location / {
      set $upstream upstream_production;
      proxy_pass http://$upstream;
      proxy_set_header x-forwarded-host $host;
      proxy_set_header Host ggn-nginx-test-target.herokuapp.com;
    }

  }
}
```

You should now be able to hit https://ggn-nginx-router-test.herokuapp.com and get the app living at https://ggn-nginx-test-target.herokuapp.com delivered to you.

**Setup nginx-test-blog**

Set up your working directory and create a `.gitignore` file

```
☯ git init .
☯ heroku create ggn-nginx-test-target
```

```
☯ cat .gitignore 
/vendor/
```

Create a `Procfile`:

```
☯ cat procfile
web: vendor/bin/heroku-php-apache2
```

Create a `composer.json` :

```
☯ cat composer.json 
{
  "require" : {
  },
  "require-dev": {
    "heroku/heroku-buildpack-php": "*"
  }
}
```

Run `composer update`

Create an `index.php`:

```
☯ cat index.php 
<h1>This is a Blog!<br />
Requestor Headers:</h1>
<br />
<?php 

foreach (getallheaders() as $name => $value) {
    echo "$name: $value\n<br />";
}
?>
```

Adding in additonal routes is as simple as setting up new location blocks and their corresponding upstream blocks. For example you could add

```
  upstream upstream_blog {
    server ggn-nginx-test-blog.herokuapp.com;
  }
```

and

```
    location /blog/ {
      set $upstream upstream_blog;
      proxy_pass http://$upstream/;
      proxy_set_header x-forwarded-host $host;
      proxy_set_header Host ggn-nginx-test-blog.herokuapp.com;
    }
```

Now when you visit https://ggn-nginx-router-test.herokuapp.com/blog/ you will get the contents of https://ggn-nginx-router-blog.herokuapp.com/

In the config directory of this repo is a sample `nginx.conf.erb` which enables this exact example.
