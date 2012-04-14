# Nginx configuration for Chive

## Introduction 

   This is a nginx configuration for running [Chive](http://chive-project.com "Chive").
   
   **Chive** is a next generation tool for managing a MySQL database
   through a web interface. 
   
   It's much better than phpMyAdmin, be it in terms of functionality
   and user experience, no to mention security. phpMyAdmin it's
   problably one of the most insecure web apps out there.
   
   It assumes that the domain affect to Chive is `chive.example.com`.

   Change accordingly to reflect your server setup.
   
   The configuration is splitted in secure (https) and standard
   (http).
   
   The first is given in the file `secure.chive.example.com.conf`. The
   second in `chive.example.com.conf` of the `sites-available` directory

## Features

   1. Filtering of invalid HTTP `Host` headers.

   2. Access to Chive is protected using
      [HTTP Basic Auth](http://wiki.nginx.org/NginxHttpAuthBasicModule
      "Basic Auth Nginx Module").  

   3. Protection of all directories emulating the `.htaccess` files
      that come with Chive.

   4. Faster and more secure handling of PHP FastCGI by Nginx using
      named groups in regular expressions instead of using
      [fastcgi_split\_path\_info](http://wiki.nginx.org/HttpFcgiModule#fastcgi_split_path_info
      "FastCGI split path info"). Requires Nginx version &ge; 0.8.25.

   5. Expire header for static assets set to the maximum.
   
   6. SSL/TLS configuration makes use of
      [Strict Transport Security](http://www.cromium.org/sts "STS")
      for protecting against MiTM attacks like
      [sslstrip](http://www.thoughtcrime.org/software/sslstrip/ "SSL strip script").
   7. IPv6 and IPv4 support.
   
   8. Possibility of using **Apache** as a backend for dealing with
      PHP. Meaning using Nginx as
      [reverse proxy](http://wiki.nginx.org/HttpProxyModule "Nginx Proxy Module").

## Basic Auth and HTTPS

   The **recommended** way to run Chive is using https. Basic Auth is
   **insecure** because the password can be sniffed on the wire.

   Ideally you should use approved Certificate Authorities issued TLS
   certificates. But if not then self signed certificates are
   fine. You just have to accept the exception in your browser. 
   
   If you're on Debian there's a `make-ssl-cert(8)` command for
   creating self signed certificates. It's included in the
   [ssl-cert](http://packages.debian.org/sid/ssl-cert "ssl-cert debian
   pkg") package.
   
   If you're on Debian or any of its derivatives like Ubuntu you need
   either the
   [thttpd-util](http://packages.debian.org/search?keywords=thttpd-util)
   or
   [apache2-utils](http://packages.debian.org/search?suite%3Dall&section%3Dall&arch%3Dany&searchon%3Dnames&keywords%3Dapache2-utils)
   package installed. 
   
   With `thttpd-util` create your password file by issuing:
   
          thtpasswd -c .htpasswd-users <user> <password>
   
   With `apache2-utils` create your password file by issuing:

          htpasswd -d -b -c .htpasswd-users <user> <password>

   You should delete this command from your shell history
   afterwards with `history -d <command number>` or alternatively
   omit the `-b` switch, then you'll be prompted for the password.

   This creates the file (there's a `-c` switch). For adding
   additional users omit the `-c`.

   Of course you can rename the password file to whatever you want,
   then accordingly change its name in the virtual host config
   file, `chive.example.com.conf` or `secure.chive.example.com.conf`.
   
## Nginx as a Reverse Proxy: Proxying to Apache for PHP

   If you **absolutely need** to use the rather _bad habit_ of
   deploying web apps relying on `.htaccess`, or you just want to use
   Nginx as a reverse proxy. The config allows you to do so. Note that
   this provides some benefits over using only Apache, since Nginx is
   much faster than Apache. Furthermore you can use the proxy cache
   and/or use Nginx as a load balancer.    

## IPv6 and IPv4

The configuration of the example vhosts uses **separate** sockets for
IPv6 and IPv4. This way is simpler for those not (yet) having IPv6
support to disable it by commenting out the
[`listen`](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen)
directive with the `ipv6only=on` parameter.

Note that the IPv6 address uses an IP _stolen_ from the
[IPv6 Wikipedia page](https://en.wikipedia.org/wiki/IPv6). You **must
replace** the indicated address by **your** address.

## Installation

   1. Move the old `/etc/nginx` directory to `/etc/nginx.old`.
   
   2. Clone the git repository from github:
   
      `git clone https://github.com/perusio/chive-nginx.git`
   
   3. Edit the `sites-available/chive.example.com.conf` or
      `sites-available/secure.chive.example.com.conf` configuration file to
      suit your requirements. Namely replacing chive.example.com with
      **your** domain.
   
   4. Setup the PHP handling method. It can be:
   
      + Upstream HTTP server like Apache with mod_php. To use this
        method comment out the `include upstream_phpcgi.conf;`
        line in `nginx.conf` and uncomment the lines:
        
            include reverse_proxy.conf;
            include upstream_phpapache.conf;

        Now you must set the proper address and port for your
        backend(s) in the `upstream_phpapache.conf`. By default it
        assumes the loopback `127.0.0.1` interface on port
        `8080`. Adjust accordingly to reflect your setup.

        Comment out **all**  `fastcgi_pass` directives in either
        `drupal_boost.conf` or `drupal_boost_drush.conf`, depending
        which config layout you're using. Uncomment out all the
        `proxy_pass` directives. They have a comment around them,
        stating these instructions.
      
      + FastCGI process using php-cgi. In this case an
        [init script](https://github.com/perusio/php-fastcgi-debian-script
        "Init script for php-cgi") is
        required. This is how the server is configured out of the
        box. It uses UNIX sockets. You can use TCP sockets if you prefer.
      
      + [PHP FPM](http://www.php-fpm.org "PHP FPM"), this requires you
        to configure your fpm setup, in Debian/Ubuntu this is done in
        the `/etc/php5/fpm` directory.
        
        Look [here](https://github.com/perusio/php-fpm-example-config) for
        an **example configuration** of `php-fpm`.
        
      Check that the socket is properly created and is listening. This
      can be done with `netstat`, like this for UNIX sockets:
      
        `netstat --unix -l`
      
      or like this for TCP sockets:
      
        `netstat -t -l`
   
      It should display the PHP CGI socket.
      
      Note that the default socket type is UNIX and the config assumes
      it to be listening on `unix:/tmp/php-cgi/php-cgi.socket`, if
      using the `php-cgi`, or in `unix:/var/run/php-fpm.sock` using
      `php-fpm` and that you should **change** to reflect your setup
      by editing `upstream_phpcgi.conf`.
   
   5. Create the `/etc/nginx/sites-enabled` directory and enable the
      virtual host using one of the methods described below.
    
      Note that if you're using the
      [nginx_ensite](http://github.com/perusio/nginx_ensite) script
      described below it **creates** the `/etc/nginx/sites-enabled`
      directory if it doesn't exist the first time you run it for
      enabling a site.
    
   6. Reload Nginx:
   
      `/etc/init.d/nginx reload`
   
   7. Check that Chive is working by visiting the configured site
      in your browser.
   
   8. Remove the `/etc/nginx.old` directory.
   
   9. Done.
   
## Enabling and Disabling Virtual Hosts

   I've created a shell script
   [nginx_ensite](http://github.com/perusio/nginx_ensite) that lives
   here on github for quick enabling and disabling of virtual hosts.
   
   If you're not using that script then you have to **manually**
   create the symlinks from `sites-enabled` to `sites-available`. Only
   the virtual hosts configured in `sites-enabled` will be available
   for Nginx to serve.
   
   
## Getting the latest Nginx packaged for Debian or Ubuntu

   I maintain a [debian repository](http://debian.perusio.net/unstable
   "my debian repo") with the
   [latest](http://nginx.org/en/download.html "Nginx source download")
   version of Nginx. This is packaged for Debian **unstable** or
   **testing**. The instructions for using the repository are
   presented on this [page](http://debian.perusio.net/debian.html
   "Repository instructions").
 
   It may work or not on Ubuntu. Since Ubuntu seems to appreciate more
   finding semi-witty names for their releases instead of making clear
   what's the status of the software included. Is it **stable**? Is it
   **testing**? Is it **unstable**? The package may work with your
   currently installed environment or not. I don't have the faintest
   idea which release to advise. So you're on your own. Generally the
   APT machinery will sort out for you any dependencies issues that
   might exist.

## Running Chive in a subdirectory instead of a subdomain

   You can run Chive in a subdirectory instead of a subdomain. Suppose
   that you run Chive in `priv/chive`. The config is:
       
       location /priv {
          ## Access is restricted using Basic Auth.
          auth_basic "Restricted Access"; # auth realm  
          auth_basic_user_file .htpasswd-users; # htpasswd file
    
          ## Chive configuration.
          location /priv/chive {

             ## Use PATH_INFO for translating the requests to the
             ## FastCGI. This config follows Igor's suggestion here:
             ## http://forum.nginx.org/read.php?2,124378,124582.
             ## This is preferable to using:
             ## fastcgi_split_path_info ^(.+\.php)(.*)$
             ## It saves one regex in the location. Hence it's faster.
             location ~ ^(?<script>.+\.php)(?<path_info>.*)$ {
                include fastcgi.conf;
                ## The fastcgi_params must be redefined from the ones
                ## given in fastcgi.conf. No longer standard names
                ## but arbitrary: named patterns in regex.
                fastcgi_param SCRIPT_FILENAME $document_root$script;
                fastcgi_param SCRIPT_NAME $script;
                fastcgi_param PATH_INFO $path_info;
                ## Passing the request upstream to the FastCGI
                ## listener.
                fastcgi_pass unix:/tmp/php-cgi/php-cgi.socket;
             }
        
             ## Protect these locations. Replicating the .htaccess
             ## rules throughout the chive distro.
             location /priv/chive/protected {
                internal;
             }
             
             location /priv/chive/yii {
                internal;
             }
           }
        }

## My other Nginx configs on github
        
   + [Drupal](https://github.com/perusio/drupal-with-nginx "Drupal
     Nginx configuration")
          
   + [WordPress](https://github.com/perusio/wordpress-nginx "WordPress Nginx
     configuration")
     
   + [Piwik](https://github.com/perusio/piwik-nginx "Piwik Nginx
     configuration")

   + [Redmine](https://github.com/perusio/redmine-nginx "Redmine Nginx
     configuration")

   + [SquirrelMail](https://github.com/perusio/squirrelmail-nginx
     "SquirrelMail Nginx configuration")

## Securing your PHP configuration

   I have created a small shell script that parses your `php.ini` and
   sets a sane environment, be it for **development** or
   **production** settings. 
   
   Grab it [here](https://github.com/perusio/php-ini-cleanup "PHP
   cleanup script").
