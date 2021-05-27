# JupyterLab with Docker & Apache Reverse Proxy
Running JupyterLab with Docker behind and Apache Reverse Proxy

I've run JupyterLab for a while now, but have recently shifted to using Docker for most of my projects. After setting up the basic container, it opened right up using the installation IP and port assigned during setup. After reestablishing notebook trust, I was ready to roll; or so I thought.  

Everything seemed to work when I browsed to the proxied web address except that the Python Kernel would not connect. After fiddling around and digging through what seemed like every possible suggestion, the simplest solution was the one that worked.  

For a basic setup rundown, I have my primary web server on a Raspberry Pi and my JupyterLab set up on a machine that runs all my Docker containers. So, my SSL certificates and all the other basic web parts are on the Pi. Everything else is proxied in from Docker. My solution to establish a kernel connection had to do with the WebSockets. Apache requires that you specifically address WebSockets when you proxy a web service in. This is unique to Apache, as other web servers that support WebSockets will automatically perform this task for you. (The WebSocket API makes it possible to open a two-way interactive communication session between the user's browser and a server. With this, you can send messages to a server and receive event-driven responses. This is critical for JupyterLab.)  

> Important thing to note: The entry for WebSockets should be before the main proxy statements.  

So without further ado, here's the solution that worked for me.  
  
```
<VirtualHost *:443>
#JupyterLab  Proxy to OMV
        ServerName jupyter.domain.com
        ServerAdmin user@domain.com

        ProxyPreserveHost On
        ProxyRequests off

        ProxyPass /api/kernels/ ws://192.168.1.186:8888/api/kernels/
        ProxyPassReverse /api/kernels/ http://192.168.1.186:8888/api/kernels/

        ProxyPass "/" "http://192.168.1.186:8888/"
        ProxyPassReverse "/" "http://192.168.1.186:8888/"
          LogLevel Debug

#               DocumentRoot /var/www/html
#                DocumentRoot /mnt/8tb/_webdir

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        #   SSL Engine Switch:
        #   Enable/Disable SSL for this virtual host.
        SSLEngine on

        <FilesMatch "\.(cgi|shtml|phtml|php)$">
                        SSLOptions +StdEnvVars
        </FilesMatch>
    <Directory /usr/lib/cgi-bin>
                    SSLOptions +StdEnvVars
    </Directory>

        #added by certbot 8 March 2021 - MFO
        Include /etc/letsencrypt/options-ssl-apache.conf
        SSLCertificateFile /etc/letsencrypt/live/domain.com/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/domain.com/privkey.pem
</VirtualHost>
```
