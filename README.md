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

2. Create a configuration in /etc/nginx/sites-available/ directory:

```bash
   sudo touch /etc/nginx/sites-available/medium_proxy.conf
   // then open and edit 
   sudo nano /etc/nginx/sites-available/medium_proxy.conf
```

3. Input your configuration:

```bash
server {
    listen 80; # or any other port you prefer

    location /medium-proxy/ {
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,>
            add_header 'Access-Control-Max-Age' 3600;
            add_header 'Content-Type' 'text/plain; charset=utf-8';
            add_header 'Content-Length' 0;
            return 204;  # Respond with status 204 No Content for OPTIONS requests
        }

        # Ensure that the proxy_pass URL is correct
        proxy_pass https://medium.com/feed/Guyycodes;  # Correct URL for the Medium feed
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        add_header 'Access-Control-Allow-Origin' '*' always;  # Add CORS header for non-OPTIONS requests
    }
}
```
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
```
// create the log file
sudo touch /var/log/nginx/stream_access.log

// you may have to adjust this depending on the user
sudo chown www-data:www-data /var/log/nginx/stream_access.log

sudo systemctl reload nginx
sudo systemctl restart nginx

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
