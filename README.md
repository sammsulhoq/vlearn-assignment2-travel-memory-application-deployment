# vlearn-assignment2-travel-memory-application-deployment
**Deployment Architecture Diagram**

Below is the structural layout of the deployment. Traffic flows from the client through Cloudflare DNS, hits the AWS Load Balancer, and is distributed across multiple EC2 instances running Nginx, Frontend (React), and Backend (Node.js) server processes.

<img width="410" height="691" alt="image" src="https://github.com/user-attachments/assets/2c606e74-db45-4975-8550-8c1fb5f18f3d" />
---

**Step-by-Step Deployment**

1. Created an Ubuntu 24.04 LTS EC2 instance on AWS. Configured the Security Group to allow inbound traffic on ports 80 (HTTP), 443 (HTTPS), and 22 (SSH).

2. Did SSH into the new instance and installed the required dependencies:
   ```
   sudo apt update && sudo apt upgrade -y
   sudo apt install nodejs npm nginx git -y
   sudo npm install -g pm2
   ```
   
3. Clones the project repository:
   ```
   git clone https://github.com/UnpredictablePrashant/TravelMemory.git
   cd TravelMemory
   ```
   
4. Navigated to the backend directory and install packages:
   ```
   cd backend
   npm install
   ```

5. Created the .env file:
   ```
   nano .env
   ```

   Added the following values in the .env file:
   ```
   PORT=3000
   MONGO_URI=mongodb+srv://<username>:<password>@cluster.mongodb.net/travelmemory?retryWrites=true&w=majority
   ```

6. Started the backend server using PM2 to ensure it runs continuously in the background:
   ```
   pm2 start index.js --name "travel-backend"
   ```

7. Configured Nginx as a reverse proxy for the backend running on port 3000:
   ```
   sudo nano /etc/nginx/sites-available/default
   ```

   Modified the location `/api` block (or root block depending on your routing setup) to route traffic to the backend
   ```
   server {
       listen 80;
       server_name _;

       location /api/ {
           proxy_pass http://localhost:3000/;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }

       # Frontend static files routing
       location / {
           root /var/www/html/frontend/dist; # Path to your compiled frontend
           index index.html;
           try_files $uri $uri/ /index.html;
       }
   }
   ```

8. Restarted Nginx:
   ```
   sudo nginx -t
   sudo systemctl restart nginx
   ```

9. Then navigated to the frontend directory:
   ```
   cd ../frontend
   ```

10. Installed dependencies:
    ```
    npm install
    npm run build
    ```

11. Move the compiled production build to the Nginx public directory:
    ```
    sudo mkdir -p /var/www/html/frontend
    sudo cp -r dist /var/www/html/frontend/
    ```

**Scaling and Load Balancing**
To satisfy high availability and scalability requirements, multiple web server instances were deployed with the following actions:

1.  An Amazon Machine Image (AMI) was created from the fully configured base EC2 instance.
2.  A second EC2 instance was launched using this AMI, replicating the exact environment effortlessly.
3.  Then created a Target Group containing both EC2 instances targeting port 80.
4.  An AWS ALB was provisioned to listen to public HTTP/HTTPS traffic and distribute requests evenly across both active instances.

**Domain Configuration with Cloudflare**
Domain name routing and security mapping are managed externally through Cloudflare DNS setup parameters:

1. Record Type: CNAME
   Name: www
   Target: <aws-alb-dns-name>.amazonaws.com	
   Proxy Status: Proxied
2. Record Type: A
   Name: @
   Target: <primary_ec2_public_ip>	
   Proxy Status: Proxied
