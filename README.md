# Assignment3
VCC3
#!/bin/bash

# Configuration
GCP_ZONE="us-central1-a"
LOCAL_DATA_PATH="$HOME/local/data"  
REMOTE_DATA_PATH="$HOME" 
GCP_USER="vboxuser"  
CPU_THRESHOLD=75
APP_DIR="myapp"
PORT="8080"

# Authenticate with GCP using your service account
gcloud auth activate-service-account --key-file="$HOME/assignment3-454611.json"
gcloud config set project assignment-3-454611

# Enable Required Services
gcloud services enable compute.googleapis.com
gcloud services enable iam.googleapis.com

# Get CPU Usage
CPU_USAGE=$(top -bn 1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
echo "CPU Usage: $CPU_USAGE%"

# Check if VM instance exists
INSTANCE_NAME=$(gcloud compute instance-groups managed list-instances scaled-vm-group --zone $GCP_ZONE --format="value(name)" | head -n 1)

# If CPU usage exceeds threshold and no VM exists, create a new VM
if (( $(echo "$CPU_USAGE > $CPU_THRESHOLD" | bc -l) )); then
    if [ -z "$INSTANCE_NAME" ]; then
        echo "CPU usage exceeded $CPU_THRESHOLD%. Creating a new VM in GCP..."

        gcloud compute instance-templates create scaled-vm-template \
            --image-family ubuntu-2204-lts \
            --image-project ubuntu-os-cloud \
            --machine-type e2-medium \
            --boot-disk-size 10GB \
            --tags http-server,https-server

        gcloud compute instance-groups managed create scaled-vm-group \
            --base-instance-name scaled-vm \
            --template scaled-vm-template \
            --size 1 \
            --zone $GCP_ZONE

        gcloud compute instance-groups managed set-autoscaling scaled-vm-group \
            --max-num-replicas 5 \
            --min-num-replicas 1 \
            --target-cpu-utilization 0.75 \
            --cool-down-period 60 \
            --zone $GCP_ZONE
        
        GCP_VM_NAME=$(gcloud compute instance-groups managed list-instances scaled-vm-group --zone $GCP_ZONE --format="value(name)" | head -n 1)
        
        echo "Waiting for VM to be ready..."
        sleep 30
        
        echo "Transferring data to GCP VM..."
        gcloud compute scp --recurse "$LOCAL_DATA_PATH" "$GCP_USER@$GCP_VM_NAME:$REMOTE_DATA_PATH" --zone="$GCP_ZONE"
            
        echo "Deploying the Node.js application..."
        gcloud compute ssh $GCP_VM_NAME --zone $GCP_ZONE --command "
          sudo apt update
          sudo apt install -y nodejs npm nginx
          
          mkdir -p /home/$USER/$APP_DIR
          cd /home/$USER/$APP_DIR
          npm init -y
          npm install express
          
          echo \"const express = require('express'); const app = express(); const port = $PORT; app.get('/', (req, res) => { res.send('Hello, world! This is a sample app deployed on $GCP_VM_NAME.'); }); app.listen(port, () => { console.log('App listening on port ' + port); });\" > app.js
          
          nohup node app.js &
          
          sudo rm /etc/nginx/sites-enabled/default
          echo 'server {
            listen 80;
            server_name _;
            location / {
              proxy_pass http://localhost:$PORT;
              proxy_http_version 1.1;
              proxy_set_header Upgrade \$http_upgrade;
              proxy_set_header Connection \"upgrade\";
              proxy_set_header Host \$host;
              proxy_cache_bypass \$http_upgrade;
            }
          }' | sudo tee /etc/nginx/sites-available/default
          
          sudo systemctl restart nginx
        "
        
        echo "Sample application deployed and running on $GCP_VM_NAME."
        
        echo "Testing auto-scaling by generating CPU load..."
        gcloud compute ssh $GCP_VM_NAME --zone $GCP_ZONE --command "
          sudo apt install -y apache2-utils
          ab -n 1000 -c 100 http://localhost:$PORT/
        "
        
        echo "Auto-scaling test completed."
        
        echo "Automation Complete!"
            
    else
        echo "GCP VM already exists. Skipping creation."
    fi

# If CPU usage drops below threshold and VM exists, shut down the VM
elif (( $(echo "$CPU_USAGE < $CPU_THRESHOLD" | bc -l) )); then
    if [ -n "$INSTANCE_NAME" ]; then
        echo "CPU usage dropped below $CPU_THRESHOLD%. Shutting down GCP VM..."
        gcloud compute instance-groups managed delete scaled-vm-group --zone $GCP_ZONE 
    else
        echo "No active GCP VM to shut down."
    fi
fi
