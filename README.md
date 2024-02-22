# Nginx-ReverseProxy

## Using NGINX as a reverse proxy

We are going to use NGINX inside an EC2 to redirect traffic between two different clusters. Cluster A is in the same AWS acccount as the EC2 machine, while cluster B is in another account. Cluster A has ***ISTIO*** a service mesh that controls all incoming traffic inside the cluster, that's why we are going to have to work outside the cluster. Let's get started

## 1. Create the instance 

Now, there are many ways you can create the EC2 instance: using the console, aws cli or terraform (to name a few). Whichever way is easier for you. We will not cover that in this runbook. Some things to keep in mind: 

* The size of this machine will depend on what kind of redirection you will use and the ammount of traffic you expect. Since this client's app doesnt handle heavy traffic and we will use a 301 (*permanent*) redirect (*which means the after the first request, all following request will go directly to the new target*), it means that we can work with a micro instance.
* You should use an Elastic IP Address to make sure the IP doesnt change and your app stops working.
* Dont forget to create a pem key to ssh into the machine (or use one you already own). It will make the process of configuring the nginx server a lot smoother. 


### Install NGINX 

Once your instance is up, connect to it. 

Update packages 

```
sudo apt-get update 
```

Install Nginx: 

```
sudo apt-get nginx
```
## 3. Configuring the server as a Reverse Proxy

Open the NGINX configuration file (/etc/nginx/nginx.conf) or create a new configuration file (e.g., /etc/nginx/conf.d/rev-proxy.conf).

For the example below, I created a new file, and configured as such: 

```
    server {
        server_name api-stage.farmarcas.com.br *.api-stage.farmarcas.com.br;

        error_log  /var/log/nginx/error.log;
        access_log  /var/log/nginx/access.log;
        
        location /orbita {
          return 301 https://orbita.api.radar.farmarcas.com.br;
        }

        location / {
     	  proxy_set_header        Host api-stage.farmarcas.com.br;
     	  proxy_set_header        Content-Type application/json;
   #  	  add_header              Cache-Control "no-store, max-age=off";
   #  	  proxy_cache_bypass      $cookie_nocache $arg_nocache;
     	  proxy_pass              https://dualstack.a63b5e44df667420286194d08671b18b-713892268.us-east-2.elb.amazonaws.com;
     	  proxy_redirect off;
     	  proxy_set_header X-Real-IP $remote_addr;
     	  proxy_ssl_server_name on;
    }

        location /nginx_status {
          stub_status on;
        }


    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/api-stage.farmarcas.com.br/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/api-stage.farmarcas.com.br/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
    server {
    if ($host = www.api-stage.farmarcas.com.br) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    if ($host = api-stage.farmarcas.com.br) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    if ($host = *.api-stage.farmarcas.com.br) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


        server_name api-stage.farmarcas.com.br *.api-stage.farmarcas.com.br;
    
    listen 80 ;
    return 404; # managed by Certbot

}

``` 

### Lets go over what each block in this file does: 

**server {}** - This first server will handle all HTTPS requests to the host and will redirect them accordingly.

**server_name api-stage.farmarcas.com.br \*\.api-stage.farmarcas.com.br**; - Defines the server names for which this block will handle requests.

**error_log  /var/log/nginx/error.log; access_log  /var/log/nginx/access.log;** - This block specifies the paths for error and access logs.

**location /orbita { return 301 https://orbita.api.radar.farmarcas.com.br;}** - Redirects requests with the URI path */orbita* to the new cluster (cluster B). It uses the status 301 (moved permanently) so future requests will go straight to the target.

**location / {}** - This large block defines the target for all other requests, it sets up a reverse proxy to forward requests to the load balancer of cluster A (the current cluster). The target of this redirect is defined in *proxy_pass*, which we will see below. 

**proxy_set_header Host api-stage.farmarcas.com.br;** - This directive sets the *Host* header that will be sent to the backend server when proxying the request. It s the hostname of the backend server that the request will be forwarded to.

**proxy_set_header Content-Type application/json;** - This directive sets the *Content-Type* header that will be sent to the backend server. It specifies that the content type of the request being forwarded is JSON.

**proxy_pass              https://dualstack.a63b5e44df667420286194d08671b18b-713892268.us-east-2.elb.amazonaws.com;** - This directive specifies the backend server to which Nginx should forward the request. In this case, it's a load balancer URL 

**proxy_redirect off;** - This directive disables the automatic rewriting of the *Location* and *Refresh* headers in the response from the backend server. It ensures that the original headers sent by the backend server are not modified.

**proxy_set_header X-Real-IP $remote_addr;** - This directive sets the X-Real-IP header to the client's IP address (*$remote_addr*). This header is useful for passing the actual client IP address to the backend server, especially when Nginx is behind a proxy.

**proxy_ssl_server_name on;** - This directive enables sending the requested server name (the value of the *Host* header) to the proxy server during SSL handshake. It's useful when the backend server expects the requested server name to be provided during SSL negotiation.

**location /nginx_status {stub_status on;}** - This directive block is used to configure the Nginx status module, which provides access to basic server metrics and status information. 

Now, the second block of the conf file is managed by certbot. This step is to show you how to install it and run it. 

After you have configured the first block (with your host and target servers). Let's install and run the certbot: 
### The next few blocks are managed by certbot, let's see what they do:

**listen 443 ssl;** - This directive tells Nginx to listen for *HTTPS* connections on port *443*.

**ssl_certificate /etc/letsencrypt/live/api-stage.farmarcas.com.br/fullchain.pem;** - This directive specifies the path to the SSL certificate file used for the domain given.The certificate contains the public key and is used to establish the server's identity during the SSL handshake.

**ssl_certificate_key /etc/letsencrypt/live/api-stage.farmarcas.com.br/privkey.pem;** - This directive specifies the path to the private key file corresponding to the SSL certificate. The private key is used for decrypting incoming encrypted data and should be kept secure.

**include /etc/letsencrypt/options-ssl-nginx.conf;** - This directive includes additional SSL/TLS configuration options from the specified file. 

**ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;** - This directive specifies the path to the file containing Diffie-Hellman parameters used for key exchange during the SSL handshake.



### Now to the second server block:

**server {}*** - This second server block handles HTTP request to the host, and redirects them accordingly:
 
**if ($host = www.api-stage.farmarcas.com.br) {return 301 https://$host$request_uri;}** - This block checks if the incoming request's host header matches www.api-stage.farmarcas.com.br. If it does, it returns a 301 (Permanent Redirect) response, redirecting the client to the HTTPS version of the same URL.

**if ($host = api-stage.farmarcas.com.br) {return 301 https://$host$request_uri;}** - Similar to the first block, this one checks if the incoming request's host header matches api-stage.farmarcas.com.br. If it does, it also returns a 301 response, redirecting the client to the HTTPS version of the same URL.

**if ($host = *.api-stage.farmarcas.com.br) {return 301 https://$host$request_uri;}** - This block employs a wildcard (\*\) to match any subdomain of api-stage.farmarcas.com.br *(such as www.api-stage.farmarcas.com.br)*. If the request matches any subdomain under api-stage.farmarcas.com.br, it returns a 301 response, redirecting the client to the HTTPS version of the same URL.

**server_name api-stage.farmarcas.com.br *.api-stage.farmarcas.com.br;** - This line specifies the server name for this block. It declares that the server should respond to requests for api-stage.farmarcas.com.br and any subdomain under api-stage.farmarcas.com.br.
    
**listen 80 ;** - This directive tells Nginx to listen for incoming connections on port 80, which is the default HTTP port.

**return 404;** - This line returns an HTTP 404 (Not Found) response to any request that reaches this block. Essentially, it indicates that if a request doesn't match any of the previous conditions, it will result in a 404 response.
}

## 4. Run Certbot to certify your hosts 

Now, the second block of the conf file is managed by certbot. This step is to show you how to install it and run it. 

After you have configured the first block (with your host and target servers). Let's install and run the certbot: 

#### Installing the certbot 

Access the instance with the nginx, and install the certbot: 

```
sudo apt install certbot python3-certbot-nginx
```
#### Run the certbot

```
sudo certbot --nginx -d <yourhost.com.br> -d <www.yourhost.com.br>
```
#### Check the file to see if worked according to plan

```
cat /etc/nginx/conf.d/<yourfile>
```

#### Restart Nginx
```
systemctl restart nginx
```

You now have an SSL certificate for HTTPS requests. Now these certificates last for about 3 months. Next, we're going to create a cronjob to renew the certificate every 3 months, just so there isn't any interference with our app. 

If you have questions or want to read further on the certbot, check the link below: 

[certbot](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-20-04)

## 5. Create a cronjob

First, lets check to see if there are any cronjobs running already (you might have to change to sudo mode for this)

```
crontab -l 
```
If you have any cronjobs, it will open the crontab file and you will be able to see them, if there are no cronjobs, you will get the response: 

```
no crontab for ubuntu
```
#### Creating a cronjob

Open the crontab file: 

```
no crontab for ubuntu - using an empty one

Select an editor.  To change later, run 'select-editor'.
  1. /bin/nano        <---- easiest
  2. /usr/bin/vim.basic
  3. /usr/bin/vim.tiny
  4. /bin/ed

Choose 1-4 [1]:
```
Choose an editor of your choice, and the crontab file will open, go to the end of the file and add schedule expression that you want. In our case, we will add one for the job to the executed every 3 months, and it will look like this: 

```
0 0 19 1,4,7,10 * /usr/bin/certbot renew --dry-run
```
Save and exit, now check again for any cronjob, and now your response should be like so: 

```
# Edit this file to introduce tasks to be run by cron.

# 
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
# 
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
# 
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
# 
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
# 
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
# 
# For more information see the manual pages of crontab(5) and cron(8)
# 
# m h  dom mon dow   command
0 0 19 1,4,7,10 * /usr/bin/certbot renew --dry-run
```
Good job, now you don't have to worry about your certification anymore. 

## 5. Change the Route53 Record

Now that your Nginx proxy is configured, and you have an everlasting certification, we are ready to redirect your url to the instance. Let's go: 

*  **Go to your Route53**
*  **Select the record you will redirect** (if it's more than one, do it one at a time)
*  **Edit record**
*  **Record type** - Set to "A"
*  **Alias** - Turn off
*  **Value** - Insert your EC2 IP address (Public or your EIP)
*  **Save**

### Testing

To test your application, insert the url and any paths that you will receive a response. If you use a testing tool like Insomnia, you can use the url (and any known paths) and see if you get a status 200 OK

CONGRATULATIONS you now have a working reverse proxy!!!
