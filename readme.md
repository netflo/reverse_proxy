extract from Digital Ocean:
https://www.digitalocean.com/community/tutorials/how-to-use-apache-as-a-reverse-proxy-with-mod_proxy-on-ubuntu-16-04



Step 1 — Enabling Necessary Apache Modules

Apache has many modules bundled with it that are available but not enabled in a fresh installation. First, we'll need to enable the ones we'll use in this tutorial.

The modules we need are mod_proxy itself and several of its add-on modules, which extend its functionality to support different network protocols. Specifically, we will use:

    mod_proxy, the main proxy module Apache module for redirecting connections; it allows Apache to act as a gateway to the underlying application servers.
    mod_proxy_http, which adds support for proxying HTTP connections.
    mod_proxy_balancer and mod_lbmethod_byrequests, which add load balancing features for multiple backend servers.

To enable these four modules, execute the following commands in succession.

    sudo a2enmod proxy
    sudo a2enmod proxy_http
    sudo a2enmod proxy_balancer
    sudo a2enmod lbmethod_byrequests

To put these changes into effect, restart Apache.

    sudo systemctl restart apache2

Apache is now ready to act as a reverse proxy for HTTP requests. In the next (optional) step, we will create two very basic backend servers. These will help us verify if the configuration works properly, but if you already have your own backend application(s), you can skip to Step 3.


=== backend ===

Step 3 — Modifying the Default Configuration to Enable Reverse Proxy

In this section, we will set up the default Apache virtual host to serve as a reverse proxy for single backend server or an array of load balanced backend servers.

Note: In this tutorial, we're applying the configuration at the virtual host level. On a default installation of Apache, there is only a single, default virtual host enabled. However, you can use all those configuration fragments in other virtual hosts as well. To learn more about virtual hosts in Apache, you can read this How To Set Up Apache Virtual Hosts on Ubuntu 16.04 tutorial.

If your Apache server acts as both HTTP and HTTPS server, your reverse proxy configuration must be placed in both the HTTP and HTTPS virtual hosts. To learn more about SSL with Apache, you can read this How To Create a Self-Signed SSL Certificate for Apache in Ubuntu 16.04 tutorial.

Open the default Apache configuration file using nano or your favorite text editor.

    sudo nano /etc/apache2/sites-available/000-default.conf

Inside that file, you will find the <VirtualHost *:80> block starting on the first line. The first example below explains how to configure this block to reverse proxy for a single backend server, and the second sets up a load balanced reverse proxy for multiple backend servers.
Example 1 — Reverse Proxying a Single Backend Server

Replace all the contents within VirtualHost block with the following, so your configuration file looks like this:
/etc/apache2/sites-available/000-default.conf

<VirtualHost *:80>
    ProxyPreserveHost On

    ProxyPass / http://127.0.0.1:8080/
    ProxyPassReverse / http://127.0.0.1:8080/
</VirtualHost>

If you followed along with the example servers in Step 2, use 127.0.0.1:8080 as written in the block above. If you have your own application servers, use their addresses instead.

There are three directives here:

    ProxyPreserveHost makes Apache pass the original Host header to the backend server. This is useful, as it makes the backend server aware of the address used to access the application.
    ProxyPass is the main proxy configuration directive. In this case, it specifies that everything under the root URL (/) should be mapped to the backend server at the given address. For example, if Apache gets a request for /example, it will connect to http://your_backend_server/example and return the response to the original client.
    ProxyPassReverse should have the same configuration as ProxyPass. It tells Apache to modify the response headers from backend server. This makes sure that if the backend server returns a location redirect header, the client's browser will be redirected to the proxy address and not the backend server address, which would not work as intended.

To put these changes into effect, restart Apache.

    sudo systemctl restart apache2

Now, if you access http://your_server_ip in a web browser, you will see your backend server response instead of standard Apache welcome page. If you followed Step 2, this means you'll see Hellow world!.
Example 2 — Load Balancing Across Multiple Backend Servers

If you have multiple backend servers, a good way to distribute the traffic across them when proxying is to use load balancing features of mod_proxy.

Replace all the contents within the VirtualHost block with the following, so your configuration file looks like this:
/etc/apache2/sites-available/000-default.conf

<VirtualHost *:80>
<Proxy balancer://mycluster>
    BalancerMember http://127.0.0.1:8080
    BalancerMember http://127.0.0.1:8081
</Proxy>

    ProxyPreserveHost On

    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/
</VirtualHost>

The configuration is similar to the previous one, but instead of specifying a single backend server directly, we've used an additional Proxy block to define multiple servers. The block is named balancer://mycluster (the name can be freely altered) and consists of one or more BalancerMembers, which specify the underlying backend server addresses. The ProxyPass and ProxyPassReverse directives use the load balancer pool named mycluster instead of a specific server.

If you followed along with the example servers in Step 2, use 127.0.0.1:8080 and 127.0.0.1:8081 for the BalancerMember directives, as written in the block above. If you have your own application servers, use their addresses instead.

To put these changes into effect, restart Apache.

    sudo systemctl restart apache2

If you access http://your_server_ip in a web browser, you will see your backend servers' responses instead of the standard Apache page. If you followed Step 2, refreshing the page multiple times should show Hello world! and Howdy world!, meaning the reverse proxy worked and is load balancing between both servers.


