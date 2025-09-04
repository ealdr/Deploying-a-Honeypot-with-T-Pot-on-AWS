# Deploying-a-Honeypot-with-T-Pot-on-AWS-and-analysis


This project documents the deployment of a [T-Pot](https://github.com/telekom-security/tpotce) honeypot on an [AWS](https://aws.amazon.com/) [EC2](https://aws.amazon.com/ec2/) instance. My goal was to capture malicious traffic and analyse attack patterns.

---

## What is a Honeypot and what is T-pot?

**'A honeypot is a computer security mechanism set to detect, deflect, or, in some manner, counteract attempts at unauthorized use of information systems. Generally, a honeypot consists of data (for example, in a network site) that appears to be a legitimate part of the site which contains information or resources of value to attackers.'** - [Wikipedia](https://en.wikipedia.org/wiki/Honeypot_(computing))

**'T-Pot is the all in one, optionally distributed, multiarch (amd64, arm64) honeypot plattform, supporting 20+ honeypots and countless visualization options using the Elastic Stack, animated live attack maps'** - [T-pot](https://github.com/telekom-security/tpotce)

---

## Setup


I first logged into my AWS dashboard/ console home and went to the EC2 service. I launched a new instance and named it tpot-honeypot. 

For the operating system, I selected the Ubuntu Server 22.04 LTS on 64-bit x86, as that is what T-pot supports and was tested on. I chose the instance type `m7i-flex.large` which has 2 CPUs and 8GB of RAM and kept the default option to auto assign a public IP.

I created an encryption key pair so I could connect securely. I selected RSA as the type, `.pem` as the file format, and downloaded the file, naming it tpot-key.pem. After that, I created a new security group. For inbound rules, I added SSH on port 22 restricted to `my IP` so only I was able to SSH into the instance, I added a custom TCP rule on port 64297 open to `my IP` for the T-Pot dashboard, and I added an All TCP rule on ports 0–65535 open to 0.0.0.0/0 so the honeypots would be reachable by anyone. Outbound was left as all traffic. I also increased the volume size to 128 GB using gp3 storage because T-Pot pulls and runs many Docker containers, each with its own images, logs, and data. I then launched the instance.

---

## Environment Setup Bullet Points

- **Cloud provider**: AWS EC2  
  - AMI: Ubuntu Server 22.04 LTS  
  - Instance type: m7i-flex.large (2 vCPU, 8GB RAM)  
  - Storage: 128GB gp3
  

- **Security Groups**:
  - SSH: port 22/tcp, restricted to `my IP`  
  - Dashboard: port 64297/tcp, open to `my IP`
  - Honeypot traffic: all TCP 0–65535, open to `0.0.0.0/0`
 
 ---
    

Once it was running, I checked the public IPv4 address which I copied for later (`INSTANCE-PUBLIC-IP`). On my Windows PC, I moved the downloaded key file into my `.ssh` directory. I then fixed the key permissions using PowerShell with `icacls`, making sure only my user account had read access.

`icacls "~\.ssh\tpot-key.pem" /inheritance:r`

`icacls "~\.ssh\tpot-key.pem" /grant:r "<MY-USER>:R"`

To connect to the server, I then ran the SSH command:

`ssh -i "~\.ssh\tpot-key.pem" ubuntu@<INSTANCE-PUBLIC-IP>`

I then updated the OS by running `sudo apt update && sudo apt upgrade -y` and installed [Git](https://git-scm.com/) with `sudo apt install git -y`. 

I then cloned the T-Pot repository with `git clone https://github.com/telekom-security/tpotce.git`. To start the installation, I ran: `./install.sh -s -t h -u <MY-USERNAME> -p <MY-PASSWORD>`.

This is what each letter means:
  - `-s` = Makes the script run without asking for confirmation.
  - `-t h` = Sets the installation type to hive which is the full T-Pot installation that comes with the web interface and dashboards.
  - `-u <MY-USERNAME>` = Defines the web interface username.
  - `-p <MY-PASSWORD>` = Defines the web interface password.

The script began setting up Docker by pulling all the honeypot images and once everything completed, the script prompted me to reboot, so I ran `sudo reboot`

After it rebooted I opened by browser and went to: `https://<INSTANCE-PUBLIC-IP>`. Because T-Pot uses a self-signed certificate, I accepted the warning and continued. At the login page, I signed in with the username and password, this loaded the T-Pot dashboard, where I could see the attack map, [Elasticvue](https://elasticvue.com/), [Kibana](https://www.elastic.co/kibana), and [Spiderfoot](https://github.com/smicallef/spiderfoot).

The main tool I will be using to view and analyse the honeypot data is Kibana. I can explore the raw attack data, filter by fields such as source IP, attack type, or timestamp, and visualizations that help me understand trends.

### ****I then left the honeypot online for a few days to collect attack data.****

---

## Data Findings and Analysis

After leaving the honeypot online for 3 days, I collected the following data, which I viewed in Kibana:

### Overall Attacks
<img width="3270" height="170" alt="image" src="https://github.com/user-attachments/assets/57235f09-5c08-4914-b31f-9d47e02757b6" />



As T-pot is a bunch of multiple honeypots of different types, here is an explanation of what each one does:
  - **Cowrie** emulates SSH services and captures brute-force login attempts.
  - **Honeytrap** is the general-purpose honeypot framework which listens on multiple ports and is designed to detect and log network-based attacks like scans.
  - **Dionaea** is a malware collection honeypot and mimics services like FTP and HTTP.
  - **CiscOASA** emulates the Cisco ASA firewall VPN service.
  - **Tanner** is a web application honeypot and aims to capture SQL injections.
  - **SentryPeer** detects SIP/VoIP attacks.
  - **HOnEtyr4p** is similar to Honeytrap but specialized for more deception layers.
  - **ConPot** simulates industrial control protocols like Siemens S7
  - **Mailoney** emulates an open mail relay.
  - **Adbhoney** exposes TCP port 5555 to attract malware targeting Android devices.


I received a total of 67,000 attacks and about 72% were from Cowrie alone, which means attackers focus on SSH brute force.

---

### Attack Map

The T-pot dashboard also includes a live attack map to view incoming attacks, also known as a **'pew pew map'**. Here is one of the screenshots I captured during the three days:
<img width="2421" height="1220" alt="image" src="https://github.com/user-attachments/assets/88e06da1-273d-4ad7-9c19-2704ce60d05f" />

Each coloured node represents a different service which is currently being attacked.

---

### Ports

The attacks focused on services that give remote access or file sharing. SSH on port 22 was the biggest target for brute force logins. VNC on port 5901 had attempts of remote access control, HTTP on port 80 showed exploitation attempts against web services. These targets show attackers were trying to gain control and spread malware.

---

### Attacker Reputation

Most of the attacking traffic is tied to known attacker IPs (mainly mass scanners) and some were bots/crawlers.



<img width="616" height="243" alt="image" src="https://github.com/user-attachments/assets/9c0fbecd-9d71-4e05-9e38-b845cea36840" />

---

### Geographic Breakdown
On the first day the United States was leading with most attacks, but then on day 2 Thailand took over which I will discuss later.

<img width="615" height="238" alt="image" src="https://github.com/user-attachments/assets/86fa3a95-a513-45fc-978f-a6ef6b2c9d7d" />

Distribution shows attacks are global but concentrated in a few hotspots.

<img width="463" height="217" alt="image" src="https://github.com/user-attachments/assets/f3b24679-562a-4336-b8c4-a7f6f4881dfb" />

---

### Credentials

These two word clouds show the attempts of SSH username and password attempts. The bigger the word the more times it was tried.

<img width="2505" height="348" alt="image" src="https://github.com/user-attachments/assets/8b6bce3b-687b-4e6f-bc99-99a0e883f556" />

The most attempted password was `(empty)` which is just pressing `Enter` after it asks for a password while setting the OS up.

---

## Thailand Attack Surge on Day 2

On day 2, a large-scale attack originated from Thailand, generating 45,000 attacks across 519 unique IPs. The activity peaked between 06:00 and 12:00, it also accounted for most of traffic during the that day. **67.2% of the total attacks (67,000) came from Thailand on day 2!!**

<img width="955" height="375" alt="image" src="https://github.com/user-attachments/assets/b906e18f-996d-4a27-a1cc-80c767fa852a" />

As you can see from the bottom left **'Attacks by Destination Port Histogram'** nearly **all** attacks from Thailand were towards the SSH 22 port, which suggests thousands of brute force attacks on the Cowrie Honeypot.


These were the most attempted username and passwords used by the large scale SSH attack from Thailand:
<img width="2526" height="350" alt="image" src="https://github.com/user-attachments/assets/d48deb50-5074-4075-93f2-31e02959fc58" />

## Final Thoughts / What I Learned
I gained valuable skills in deploying and managing honeypots for monitoring malicious traffic, including setup and configuration on AWS. I developed a deep understanding of how attackers target exposed services and how to capture traffic in real time. It was also a rewarding experience that expanded my technical skill set and provided practical insights into real-world cyber threats.

##  Thanks for Reading

Thanks for checking out my Honeypot using T-pot and AWS project.
If you'd like to connect or ask questions, you can reach me via [LinkedIn](https://www.linkedin.com/in/ethan-a-a95b14324/)


