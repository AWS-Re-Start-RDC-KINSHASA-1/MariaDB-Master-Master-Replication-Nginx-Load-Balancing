# Master-Master Replication with MariaDB and Nginx Load Balancing

This tutorial outlines the steps to set up Master-Master replication with MariaDB on two Amazon EC2 instances running Amazon Linux 2. Nginx is used for load balancing the database traffic between the two instances.

## Prerequisites

- Two EC2 instances with Amazon Linux 2.
- Ubuntu server with Nginx installed.

## MariaDB Master-Master Replication Setup

1. **Install MariaDB:**
```bash
sudo yum install mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

2. **Secure MariaDB installation**
```bash
sudo mysql_secure_installation
```

3. **Configure Master-Master Replication:**
On both instances:
   - Edit MariaDB configuration file
      ```bash
      sudo nano /etc/my.cnf.d/server.cnf
      ```
   - Add the following lines:
     ```cnf
     server-id = 1  # Use 2 on the other instance
     log_bin = /var/log/mariadb/mariadb-bin
     bind-address = 0.0.0.0
     ```
   - Restart MariaDB
     ```bash
     sudo systemctl restart mariadb
     ```
   - Create replication user
     ```sql
     CREATE USER 'replication_user'@'%' IDENTIFIED BY 'your_password';
     GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
     FLUSH PRIVILEGES;
     ```
   - Get Master Status:
     Note down the current position of the binary log by running:
     ```sql
     SHOW MASTER STATUS;
     ```
     Record the values for File and Position.

4. **Configure Replication:**
   - On the first intstance:
     ```sql
     CHANGE MASTER TO
        MASTER_HOST='second_instance_ip',
        MASTER_USER='repl',
        MASTER_PASSWORD='your_password',
        MASTER_LOG_FILE='second_master_log_file',
        MASTER_LOG_POS=second_master_log_pos;
     ```
   - On the second intance:
     ```sql
     CHANGE MASTER TO
        MASTER_HOST='first_instance_ip',
        MASTER_USER='repl',
        MASTER_PASSWORD='your_password',
        MASTER_LOG_FILE='first_master_log_file',
        MASTER_LOG_POS=first_master_log_pos;
     ```
   - On both instances:
     ```sql
     START SLAVE;
     ```

5. **Verify Replication:**
   - On either instances
     ```sql
     SHOW SLAVE STATUS\G
     ```
     Ensure Slave_IO_Running and Slave_SQL_Running are both Yes.

6. **Create the load balancer user:**
```sql
CREATE USER 'load_balancer_user'@'%' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON *.* TO 'load_balancer_user'@'%' IDENTIFIED BY 'your_password' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

## Nginx Load Balancing Configuration

1. **Install Nginx on Ubuntu:**
   - Connect to your Ubuntu server and install Nginx:
     ```bash
     sudo apt update
     sudo apt install nginx
     ```

2. **Configure Nginx for Load Balancing:**
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
     Replace `<EC2_Instance_1_IP>`, `<EC2_Instance_2_IP>` and `<Port>` with your actual instance details.

3. **Test the Configuration:**
   - Check configuration syntax
     ```bash
     sudo nginx -t
     ```
   - Restart Nginx to apply changes:
     ```bash
     sudo systemctl restart nginx
     ```
   - Access your domain in a web browser to ensure load balancing is working.

5. **Health Checks (Optional):**
   - Consider adding health checks for instance availability.
  
6. **Test the Load Balancer:** 
   - Connect to the Nginx load balancer from a MySQL client, and you should be able to interact with your MariaDB databases.
     ```bash
     mysql -h <Nginx_Public_IP> -u <load_balancer_user> -p
     ```
     Replace <Nginx_Public_IP> with the public IP address of your Nginx instance.
