# Docker Networking & Nginx Load Balancer Project

## Project Objective

This project demonstrates how to host two Nginx web servers inside Docker and distribute incoming HTTP requests through an Nginx Load Balancer.

The project combines practical networking concepts such as:

* IP Address
* Subnet
* Gateway
* Docker Bridge Network
* TCP
* Ports
* Port Mapping
* DNS / Container Name Resolution
* HTTP
* Firewall / AWS Security Group
* Load Balancing
* Round Robin

---

## Architecture

```text
Internet / Browser
       |
       v
AWS Security Group
       |
       v
EC2 Ubuntu Server
       |
       v
Nginx Load Balancer :8080
       |
       v
Docker Network: web-network
       |
       +----------------------+
       |                      |
       v                      v
web-server-1             web-server-2
Nginx :80                Nginx :80
```

---

# 1. EC2 Environment

AWS EC2 instance:

* Ubuntu Server 24.04 LTS
* Instance Type: t3.micro
* Storage: 12 GiB gp3
* Public IP enabled

SSH connection was established from Git Bash.

---

# 2. Verify Linux Environment

```bash
whoami
```

```bash
cat /etc/os-release
```

```bash
df -h /
```

These commands verify the current user, Ubuntu version and disk space.

---

# 3. Install Docker

Update packages:

```bash
sudo apt update
```

Install Docker:

```bash
sudo apt install -y docker.io
```

Check Docker version:

```bash
docker --version
```

Check Docker service:

```bash
sudo systemctl status docker
```

Test Docker installation:

```bash
sudo docker run hello-world
```

Expected result:

```text
Hello from Docker!
```

---

# 4. Create Project Directory

```bash
mkdir -p ~/networking-loadbalancer-project
```

```bash
cd ~/networking-loadbalancer-project
```

Verify location:

```bash
pwd
```

---

# 5. Git Repository Setup

Initialize Git:

```bash
git init
```

Configure Git username:

```bash
git config --global user.name "Rishi Anand"
```

Configure Git email:

```bash
git config --global user.email "YOUR_GITHUB_EMAIL"
```

Check configuration:

```bash
git config --global --list
```

Add README:

```bash
git add README.md
```

Initial commit:

```bash
git commit -m "Initial project setup"
```

Check remote:

```bash
git remote -v
```

Push project to GitHub:

```bash
git push -u origin main
```

---

# 6. Create Docker Network

Create a custom Docker bridge network:

```bash
docker network create web-network
```

Check network:

```bash
docker network ls
```

Inspect network:

```bash
docker network inspect web-network
```

Observed network configuration:

```text
Subnet: 172.18.0.0/16
Gateway: 172.18.0.1
Driver: bridge
```

The custom network allows Docker containers to communicate with each other using container names.

---

# 7. Create Web Server 1

```bash
docker run -d --name web-server-1 --network web-network nginx
```

Check container:

```bash
docker ps
```

Find container IP:

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web-server-1
```

Observed IP:

```text
172.18.0.2
```

---

# 8. Create Web Server 2

```bash
docker run -d --name web-server-2 --network web-network nginx
```

Check container:

```bash
docker ps
```

Find container IP:

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web-server-2
```

Observed IP:

```text
172.18.0.3
```

Container IPs can change if containers are recreated.

---

# 9. Test Container-to-Container Communication

Test Web Server 1:

```bash
docker run --rm --network web-network curlimages/curl -s -o /dev/null -w "%{http_code}\n" http://web-server-1
```

Expected:

```text
200
```

Test Web Server 2:

```bash
docker run --rm --network web-network curlimages/curl -s -o /dev/null -w "%{http_code}\n" http://web-server-2
```

Expected:

```text
200
```

This confirms that:

* Both containers are reachable.
* Docker internal DNS resolves container names.
* HTTP communication is working.

---

# 10. Inspect Docker Network

```bash
docker network inspect web-network
```

This shows the containers connected to the network and their internal IP addresses.

---

# 11. Publish Web Server 1 Port

Initially Web Server 1 did not have a host port mapping.

Check:

```bash
docker port web-server-1
```

Recreate Web Server 1 with port mapping:

```bash
docker stop web-server-1
```

```bash
docker rm web-server-1
```

```bash
docker run -d --name web-server-1 --network web-network -p 8081:80 nginx
```

Check port mapping:

```bash
docker port web-server-1
```

Expected:

```text
80/tcp -> 0.0.0.0:8081
80/tcp -> [::]:8081
```

Here:

```text
8081 = EC2 host port
80   = Container port
```

---

# 12. Test Web Server 1 From EC2 Host

```bash
curl -I http://localhost:8081
```

Expected:

```text
HTTP/1.1 200 OK
```

---

# 13. AWS Security Group

The EC2 Security Group was configured to allow:

```text
SSH       TCP 22    My IP
Custom TCP 8081     My IP
```

SSH is required to connect to the EC2 instance.

Port 8081 allows external access to Web Server 1 for testing.

---

# 14. Test Web Server From Browser

Open:

```text
http://EC2_PUBLIC_IP:8081
```

The Nginx Welcome page was successfully displayed in the browser.

---

# 15. Customize Web Server 1 Response

```bash
docker exec web-server-1 sh -c 'echo "Hello from Web Server 1" > /usr/share/nginx/html/index.html'
```

Test:

```bash
curl http://localhost:8081
```

Expected:

```text
Hello from Web Server 1
```

---

# 16. Customize Web Server 2 Response

```bash
docker exec web-server-2 sh -c 'echo "Hello from Web Server 2" > /usr/share/nginx/html/index.html'
```

Test:

```bash
docker run --rm --network web-network curlimages/curl -s http://web-server-2
```

Expected:

```text
Hello from Web Server 2
```

Different responses make load balancing easy to observe.

---

# 17. Create Nginx Load Balancer Configuration

Create the configuration file:

```bash
vim load-balancer.conf
```

Configuration:

```nginx
events {}

http {
    upstream backend_servers {
        server web-server-1:80;
        server web-server-2:80;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend_servers;
        }
    }
}
```

The `upstream` block defines the backend servers.

The Load Balancer forwards requests to:

```text
web-server-1:80
web-server-2:80
```

---

# 18. Create Load Balancer Container

```bash
docker run -d --name load-balancer --network web-network -p 8080:80 -v "$(pwd)/load-balancer.conf:/etc/nginx/nginx.conf:ro" nginx
```

Check containers:

```bash
docker ps
```

The architecture now contains:

```text
Load Balancer :8080
       |
       +---- web-server-1 :80
       |
       +---- web-server-2 :80
```

---

# 19. Test Load Balancer

Test from EC2:

```bash
curl http://localhost:8080
```

Example response:

```text
Hello from Web Server 1
```

Run multiple requests:

```bash
for i in {1..10}; do curl -s http://localhost:8080; echo; done
```

Observed:

```text
Hello from Web Server 2
Hello from Web Server 1
Hello from Web Server 2
Hello from Web Server 1
Hello from Web Server 2
Hello from Web Server 1
Hello from Web Server 2
Hello from Web Server 1
Hello from Web Server 2
Hello from Web Server 1
```

This demonstrates Nginx's default **Round Robin Load Balancing**.

---

# 20. Port Summary

| Component               | Port |
| ----------------------- | ---: |
| Web Server 1 container  |   80 |
| Web Server 1 host       | 8081 |
| Web Server 2 container  |   80 |
| Load Balancer container |   80 |
| Load Balancer host      | 8080 |

---

# 21. Networking Concepts Used

| Concept        | Practical Usage                          |
| -------------- | ---------------------------------------- |
| IP Address     | Identified Docker containers             |
| Subnet         | `172.18.0.0/16` Docker network           |
| Gateway        | `172.18.0.1`                             |
| Bridge Network | `web-network`                            |
| TCP            | Used for HTTP connections                |
| Port           | 80, 8080 and 8081                        |
| Port Mapping   | `8081:80` and `8080:80`                  |
| DNS            | Docker resolved container names          |
| HTTP           | Nginx web traffic                        |
| Firewall       | AWS Security Group                       |
| Load Balancer  | Nginx                                    |
| Round Robin    | Requests distributed between two servers |

---

# 22. Final Architecture

```text
                     Internet
                        |
                        v
              AWS Security Group
                  TCP Port 8080
                        |
                        v
                 EC2 Ubuntu Server
                        |
                        v
             Nginx Load Balancer
                    :8080
                        |
                        v
              Docker web-network
                 /            \
                /              \
               v                v
        web-server-1       web-server-2
            :80                :80
               \              /
                \            /
                 Nginx Backend
```

---

# 23. GitHub Files

The project contains:

```text
networking-loadbalancer-project/
│
├── README.md
│
└── load-balancer.conf
```

`README.md` contains the complete project documentation and commands.

`load-balancer.conf` contains the Nginx Load Balancer configuration.

---

# 24. Project Status

Core project completed.

Successfully demonstrated:

* Docker installation
* Custom Docker network
* Two Nginx backend servers
* Container-to-container communication
* Docker DNS
* Port mapping
* AWS Security Group
* HTTP testing
* Nginx Load Balancer
* Round Robin load balancing
* Git and GitHub project management

