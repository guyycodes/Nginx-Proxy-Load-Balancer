# NGINX Proxy and Load Balancer Configuration

This repository contains a sample configuration for setting up an NGINX proxy and load balancer. NGINX is a powerful web server and reverse proxy that can be used to distribute incoming traffic across multiple backend servers, improving performance, scalability, and reliability.

## Features

- Reverse proxy functionality to forward requests to backend servers
- Load balancing capabilities to distribute traffic evenly among multiple servers
- SSL/TLS termination for secure communication between clients and the proxy
- Customizable configuration files for easy setup and management

## Prerequisites

Before using this configuration, ensure that you have the following prerequisites:

- NGINX installed on your system
- Backend servers configured and running (e.g., web servers, application servers)
- SSL/TLS certificate and key files (if enabling SSL/TLS)

## Getting Started (Proxy Server)

To get started with the NGINX proxy and load balancer configuration, follow these steps:

1. Download the full version of Nginx to your device (Im using linux commands as examples):

```bash
   sudo apt-get -y install nginx-full
```

2. Create a configuration for logging:

```bash
   // create the file for your server configurations
   sudo touch /etc/nginx/sites-available/medium_proxy.conf

   // create a file for logging
   sudo touch /var/log/nginx/proxy_access.log

   // give it permissions
   sudo chown www-data:www-data /var/log/nginx/proxy_access.log

   // setup log rotation to keep it from growing forever
   sudo nano /etc/logrotate.d/nginx_proxy

   // This setup rotates the log daily, keeps 14 days of logs, compresses old versions, and ensures proper permissions and ownership are maintained. It also sends the USR1 signal to NGINX to reopen log files after rotation.
    /var/log/nginx/proxy_access.log {
            daily                
            rotate 10            
            missingok            
            notifempty           
            compress             
            delaycompress        
            create 0640 www-data www-data 
            sharedscripts
            postrotate
                if [ -f /var/run/nginx.pid ]; then
                    kill -USR1 `cat /var/run/nginx.pid`
                fi
            endscript
    }

   // now open and edit 
   sudo nano /etc/nginx/sites-available/medium_proxy.conf

```

3. Input your configuration in /etc/nginx/sites-available/medium_proxy.conf:

```bash
server {
    listen 443 ssl;
    server_name testProxy;

    error_log /var/log/nginx/error.log debug;

    ssl_certificate /etc/ssl/certs/testProxy.crt;
    ssl_certificate_key /etc/ssl/private/testProxy.key;

    access_log /var/log/nginx/proxy_access.log;

    ssl_protocols TLSv1.2 TLSv1.3;  # Ensure using TLS protocols that are broadly compatible
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4';
    ssl_session_tickets off;  # Disable session tickets for improved security

    location /medium-proxy/ {
        proxy_pass https://medium.com/feed/@guyycodes;
        proxy_set_header Host medium.com;
        proxy_set_header User-Agent $http_user_agent;  # Forward the User-Agent from the original request
        proxy_set_header Accept $http_accept;  # Forward the Accept header from the original request
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_ssl_server_name on;  # Ensure proper SSL certificate verification by setting the server name for SSL handshake
        # Managing connections:
        proxy_http_version 1.1;  # Use HTTP/1.1 which is necessary for proper keep-alive functionality
        proxy_set_header Connection "";  # This is necessary to drop the Connection header when passing to the upstream to avoid issues with keep-alive

        proxy_ssl_verify off;  # Disable SSL verification for self-signed certificates

        proxy_cache_bypass $http_upgrade;  # Ensures that WebSocket upgrades are not cached
        proxy_cache_valid 200 302 10m;  # Cache valid responses
        proxy_cache_valid 404 1m;  # Cache 404 responses for a minute (adjust as needed)

        add_header X-Frame-Options SAMEORIGIN;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=15552000; includeSubDomains; preload";  # This enforces SSL/TLS connections
    }
}


// create a self signed certificate
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/testProxy.key -out /etc/ssl/certs/testProxy.crt


// then open up..
   sudo nano /etc/nginx/nginx.conf

// and add under the http basic settings: 
    log_format basic '$remote_addr - $remote_user [$time_local] '
                     '"$request" $status $body_bytes_sent '
                     '"$http_referer" "$http_user_agent" '
                     '$request_time';  # Time to process the request

```
4) Test the configuration: 
```bash
sudo nginx -t

// if the test is succesfull, then create a symbolic link
sudo ln -s /etc/nginx/sites-available/medium_proxy.conf /etc/nginx/sites-enabled/

// test the logging cofiguration
sudo logrotate -d /etc/logrotate.d/nginx_proxy

// test the symbolic link
ls -l /etc/nginx/sites-enabled/

// reload nginx and restart 
sudo systemctl reload nginx
sudo systemctl start nginx

// monitor the log file
tail -f /var/log/nginx/proxy_access.log

// test
curl -i -v -H "Host: testProxy.com" http://<server ip addr>/medium-proxy/



```



--- 
# Nginx as a Load Balancer 
1) ssh or log into the load balancer & install nginx
- create a new directory (WE WIL USE A TCP STREAM - A FEATURE OF NGINX)
```bash
sudo apt-get -y install nginx-full

// enable nginx to start automatically on system start
sudo systemctl enable nginx
sudo systemctl status nginx
sudo mkdir -p /etc/nginx/tcpconf.d

```
- next we will edit the main Nginx config file - use command
```bash

sudo nano /etc/nginx/nginx.conf
/// go the the very bottom of the file that opens up and add the following line:

include /etc/nginx/tcpconf.d/*;
/// this puts the tcp stream configuration file into the nginx server

```
- NOW WE NEED TO CREATE THE CONFIGURATION
- Set variables: these need to be the IP addresses of the upstream controller node(s) we will load balance our traffic to:
    * CONTROLLER0_IP=192.168.30.10
```bash

/// now set the configuration file for the tcp stream (Im calling the save file kubernetes.conf, but yu can call it what you want, just ensure you replace the name in all the relevant location demonstrated in the example)
cat << EOF | sudo tee /etc/nginx/tcpconf.d/kubernetes.conf
stream {

    # this is a pass through setup, we are not decrypting, were using Nginx as a TCP/UDP load balancertraffic so no certificates are needed

    log_format basic '$remote_addr [$time_local] '
                        '$protocol $status $bytes_sent $bytes_received '
                        '$session_time';

    upstream kubernetes {
        server $CONTROLLER0_IP:6443;

    }

    server{
        listen 6443;
        listen 443;
        proxy_pass kubernetes;
        access_log /var/log/nginx/stream_access.log basic;
    }
}
EOF
```
- Reload nginx so it can pickup the configuration
```bash
// open up using sudo nano....
sudo nano /etc/nginx/nginx.conf

// at the very bottom of the file, last item, very bottom paste in....
include /etc/nginx/tcpconf.d/*;

// save and close, then reload nginx
sudo nginx -s reload

/// test the load balancer from your local machine:
curl -k https://localhost:6443/version
// response should come from the device which Nginx is forwarding to 
// change 6443 to your desired port & localhost to the ip address of the device running Nginx
```

# setup logging for the load balancer

```bash
stream {
    log_format basic '$remote_addr [$time_local] '
                     '$protocol $status $bytes_sent $bytes_received '
                     '$session_time';

    # Your existing stream server blocks
    upstream kubernetes {
        server $CONTROLLER0_IP:6443;
    }

    server {
        listen 6443;
        listen 443;
        proxy_pass kubernetes;
        access_log /var/log/nginx/stream_access.log basic;
    }
}
```

- Make the log file and allow it to be written to: 
- update the ownership and permissions for the Nginx stream access log file
``` bash
// create the log file
sudo touch /var/log/nginx/stream_access.log

// you may have to adjust this depending on the user
sudo chown www-data:www-data /var/log/nginx/stream_access.log

sudo systemctl reload nginx
sudo systemctl restart nginx

// setup log rotation
sudo nano /etc/logrotate.d/nginx_stream

// then add this into /etc/logrotate.d/nginx_stream
 /var/log/nginx/stream_access.log {
    daily                # Rotate the log daily
    rotate 10            # Keep 10 days of backlogs
    missingok            # It's okay if the log file is missing
    notifempty           # Do not rotate the log if it is empty
    compress             # Compress (gzip) log files on rotation
    delaycompress        # Delay compression until the next rotation cycle
    create 0640 www-data www-data # Create new log files with set permissions
    sharedscripts
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}

    // enable permissions
    sudo chown www-data:www-data /var/log/nginx/stream_access.log

    // test the cofiguration
    sudo logrotate -d /etc/logrotate.d/nginx_stream

    // view dynamic logs 
    tail -f /var/log/nginx/stream_access.log

```



4. If you want to enable SSL/TLS, update the `ssl` block in the `nginx.conf` file with the paths to your SSL/TLS certificate and key files.

5. Start the NGINX service:

   ```
   sudo nginx -c /path/to/nginx.conf
   ```

   Make sure to provide the correct path to your `nginx.conf` file.

6. Access your application through the NGINX proxy by visiting the configured domain or IP address in your web browser.

## Configuration explination

The main configuration file is `nginx.conf`, which contains the essential settings for the NGINX proxy and load balancer. Here are some key sections of the configuration:

- `upstream`: Defines a group of backend servers to which the proxy will forward requests. You can specify multiple servers for load balancing.
- `server`: Configures the virtual server that listens for incoming requests. It includes settings such as the listening port, server name, and location blocks.
- `location`: Specifies how to handle requests for different URI paths. You can define proxy settings, SSL/TLS configuration, and other directives within location blocks.

Feel free to customize the configuration according to your specific requirements.

## Testing

To test the NGINX proxy and load balancer configuration, you can use tools like `curl` or a web browser. Send requests to the configured domain or IP address and verify that the requests are being properly forwarded to the backend servers.

You can also check the NGINX access and error logs to monitor the traffic and troubleshoot any issues.

## Deployment

To deploy the NGINX proxy and load balancer configuration in a production environment, follow these general steps:

1. Install NGINX on your server.
2. Copy the `nginx.conf` file from this repository to the appropriate location on your server (e.g., `/etc/nginx/nginx.conf`).
3. Customize the configuration file as needed, specifying the backend servers, SSL/TLS settings, and other relevant directives.
4. Start or restart the NGINX service to apply the configuration changes.
5. Monitor the NGINX logs and system performance to ensure smooth operation.

Remember to secure your server, keep NGINX and its dependencies up to date, and regularly monitor the system for any potential issues or security vulnerabilities.

## Contributing

Contributions to this repository are welcome. If you find any issues or have suggestions for improvements, please open an issue or submit a pull request.

## License

This project is licensed under the [MIT License](LICENSE).

## Acknowledgements

- [NGINX Documentation](https://nginx.org/en/docs/)
- [NGINX Community](https://www.nginx.com/resources/wiki/)
