# Nginx Load Balancing Setup for Amazon EC2 Instances (Ubuntu)

This repository provides a basic setup for load balancing using Nginx on Ubuntu with two Amazon EC2 instances running Amazon Linux 2.

## Prerequisites

- Two EC2 instances with Amazon Linux 2.
- Ubuntu server with Nginx installed.

## Setup Instructions

1. **Create EC2 Instances:**
   - Launch two EC2 instances in the same VPC with Amazon Linux 2.
   - Ensure that security groups allow traffic on the desired ports.

2. **Install Nginx on Ubuntu:**
   - Connect to your Ubuntu server and install Nginx:
     ```bash
     sudo apt update
     sudo apt install nginx
     ```

3. **Configure Nginx for Load Balancing:**
   - Edit the Nginx configuration file:
     ```bash
     sudo nano /etc/nginx/nginx.conf
     ```
   - Add the following configuration to enable load balancing:
     ```nginx
     stream {
         upstream backend {
             server <EC2_Instance_1_IP>:<Port>;
             server <EC2_Instance_2_IP>:<Port>;
         }

         server {
             listen 3306;
             server_name your_domain.com;

             location / {
                 proxy_pass http://backend;
                 proxy_set_header Host $host;
                 proxy_set_header X-Real-IP $remote_addr;
                 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                 proxy_set_header X-Forwarded-Proto $scheme;
             }
         }
     }
     ```
     Replace `<EC2_Instance_1_IP>`, `<EC2_Instance_2_IP>`, `<Port>`, and `your_domain.com` with your actual instance details.

4. **Test the Configuration:**
   - Restart Nginx to apply changes:
     ```bash
     sudo systemctl restart nginx
     ```
   - Access your domain in a web browser to ensure load balancing is working.

5. **Health Checks (Optional):**
   - Consider adding health checks for instance availability.
