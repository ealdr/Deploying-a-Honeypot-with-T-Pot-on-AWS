# Deploying-a-Honeypot-with-T-Pot-on-AWS
## Overview
This project documents the deployment of a [T-Pot](https://github.com/telekom-security/tpotce) honeypot on an [AWS](https://aws.amazon.com/) [EC2](https://aws.amazon.com/ec2/) instance. My goal was to capture malicious traffic and analyze attack patterns.

I first logged into my AWS dashboard/ console home and went to the EC2 service. From there, I launched a new instance and named it tpot-honeypot. 

For the operating system, I selected the Ubuntu Server 22.04 LTS AMI on 64-bit x86, as that is what the T-pot project offcially supports and was tested on. I chose the instance type `m7i-flex.large` which has 2 vCPUs and 8GB of RAM. I kept the default option to auto assign a public IP.

Next, I created an encryption key pair so I could connect securely. I selected RSA as the type, `.pem` as the file format, and downloaded the file, naming it tpot-key.pem. After that, I created a new security group. For inbound rules, I added SSH on port 22 restricted to `my IP` so only I was able to SSH into the instance, I added a custom TCP rule on port 64297 open to `my IP` for the T-Pot dashboard, and I added an All TCP rule on ports 0–65535 open to 0.0.0.0/0 so the honeypots would be reachable by anyone. Outbound was left as all traffic. I also increased the volume size to 128 GB using gp3 storage because T-Pot pulls and runs many Docker containers, each with its own images, logs, and data. I then launched the instance.

## Environment Setup

- **Cloud provider**: AWS EC2  
  - **AMI**: Ubuntu Server 22.04 LTS  
  - **Instance type**: m7i-flex.large (2 vCPU, 8GB RAM)  
  - **Storage**: 128GB gp3
  

- **Security Groups**:
  - **SSH**: port 22/tcp, restricted to `my IP`  
  - **Dashboard**: port 64297/tcp, open to `my IP`
  - **Honeypot traffic**: all TCP 0–65535, open to `0.0.0.0/0`

Once it was running, I checked the public IPv4 address which I coppied for later (`INSTANCE-PUBLIC-IP`). On my Windows PC, I moved the downloaded key file into my `.ssh` directory. I then fixed the key permissions using PowerShell with `icacls`, making sure only my user account had read access.

`icacls "~\.ssh\tpot-key.pem" /inheritance:r`

`icacls "~\.ssh\tpot-key.pem" /grant:r "<MY-USER>:R"`

To connect to the server, I then ran the SSH command:

`ssh -i "~\.ssh\tpot-key.pem" ubuntu@<INSTANCE-PUBLIC-IP>`

I accepted the fingerprint warning and was connected as the ubuntu user. Once inside, I updated the OS by running `sudo apt update && sudo apt upgrade -y` and installed [Git](https://git-scm.com/) with `sudo apt install git -y`. I also set the timezone to UTC using `timedatectl` so logs would be consistent.

I thenc loned the T-Pot repository with `git clone https://github.com/telekom-security/tpotce.git` and moved into the new directory. To start the installation, I ran: `./install.sh -s -t h -u <MY-USERNAME> -p <MY-PASSWORD>`.

This is what each letter means:
`-s` = This makes the script run without asking for confirmation.
`-t h` = Sets the installation type to hive which is the full T-Pot installation that comes with the web interface and dashboards.
`-u <MY-USERNAME>` = Defines the web interface username.
`-p <MY-PASSWORD>` = Defines the web interface password.

The script began setting up Docker by pulling all the honeypot images and once everything completed, the script prompted me to reboot, so I ran `sudo reboot`

After it rebooted I opened by browser and went to: `https://<INSTANCE-PUBLIC-IP>`. Because T-Pot uses a self-signed certificate, I accepted the warning and continued. At the login page, I signed in with the username and password, this loaded the T-Pot dashboard, where I could see the attack map, Elasticvue, Kibana, and Spiderfoot.



