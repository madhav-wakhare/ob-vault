``` bash
# events block generally contains directives that affect the connection processing model of Nginx.

events {

# Set the maximum number of simultaneous connections that can be handled by each worker process spun up by Nginx. This value can be adjusted based on the expected traffic and server capacity.

worker_connections 1024;

}

  

http {

# Define an upstream block to specify the backend API servers that Nginx will proxy requests to. This allows for load balancing and failover between multiple backend servers.

upstream api_servers {

server api1:8081;

server api2:8082;

}

  

server {

listen 8080;

  

location / {

# Proxy requests to the upstream API servers

proxy_pass http://api_servers;

# Set the Host header to the original host requested by the client

proxy_set_header Host $host;

# Set the X-Real-IP header to the client's IP address

proxy_set_header X-Real-IP $remote_addr;

# Set the X-Forwarded-For header to include the client's IP address

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

# Set the X-Forwarded-Proto header to indicate the protocol used

proxy_set_header X-Forwarded-Proto $scheme;

}

}

}
```



An upstream is simply a **group of backend servers**.
