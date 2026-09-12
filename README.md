# Need4Rides: Network and Security Summary
## BSc final project (Graded 19 out of 20)

> The entire machine network described in this document is built and managed through a set of automated Bash scripts (.sh). These scripts handle everything from creating and configuring instances to failover and scaling. For security reasons, these scripts are not publicly available, as they contain sensitive credentials and infrastructure configuration details.

## Network Architecture

The infrastructure runs on AWS in the eu-west-1 region, inside a dedicated VPC called **taxi-vpc** with CIDR block 10.0.0.0/16. The VPC is split into three subnets, each with a distinct role, following the principle of layer separation.

The **taxi-public** subnet (10.0.0.0/24) holds the components with public exposure: the bastion host and the two load balancers. The **taxi-private** subnet (10.0.1.0/24) holds the application servers, with no direct internet access. The **taxi-data** subnet (10.0.2.0/24) holds the MongoDB instances, fully isolated from outside traffic.

Inbound public traffic (HTTP/HTTPS) arrives through an **Internet Gateway** attached to the public subnet. The Internet Gateway is what allows instances with a public IP — in this case the bastion and the load balancers — to be reachable from the internet and to initiate outbound connections to it. The application servers need outbound access to external APIs (Stripe, Nominatim, OSRM, GeoAPI.pt) but cannot have a public IP, so that traffic goes through a **NAT Gateway**, also in the public subnet. The NAT Gateway performs address translation, allowing private instances to reach the internet for outbound calls without ever being directly reachable from outside. The load balancers need to talk to the AWS API to move the Elastic IP during failover, so there's an interface VPC Endpoint for the EC2 service, keeping that traffic off the public internet entirely.

Two Elastic IPs are reserved: one permanently attached to the bastion (63.33.156.38) and another to the site (54.229.43.57), which migrates between load balancers during failover.

---

## Security Groups 🕸

Four security groups exist, one per infrastructure layer. Each has minimal inbound and outbound rules, only what's strictly necessary.

### bastion-sg

The real security here comes from the private key (.pem), not from restricting the source IP.

| Direction | Port / Protocol | Source / Destination | Purpose |
|---|---|---|---|
| Inbound | TCP 22 (SSH) | 0.0.0.0/0 | Entry point for administrators |
| Outbound | TCP 22 (SSH) | loadbalancer-sg, appserver-sg, database-sg | Maintenance access to every other instance |

### loadbalancer-sg

| Direction | Port / Protocol | Source / Destination | Purpose |
|---|---|---|---|
| Inbound | TCP 80 (HTTP) | 0.0.0.0/0 | Public web traffic |
| Inbound | TCP 443 (HTTPS) | 0.0.0.0/0 | Public web traffic |
| Inbound | TCP 22 (SSH) | bastion-sg | Maintenance access |
| Outbound | TCP 3000 | appserver-sg | Nginx proxying to application servers |
| Outbound | TCP 443 (HTTPS) | 10.0.0.0/16 | AWS VPC Endpoint, moves the Elastic IP during failover |
| Outbound | Protocol 112 (VRRP) | loadbalancer-sg | Keepalived communication between the two load balancers (112 is the IP protocol number, not a TCP/UDP port) |

### appserver-sg

| Direction | Port / Protocol | Source / Destination | Purpose |
|---|---|---|---|
| Inbound | TCP 3000 | loadbalancer-sg | Application traffic from Nginx |
| Inbound | TCP 22 (SSH) | bastion-sg | Maintenance access |
| Outbound | TCP 27017 | database-sg | MongoDB connection |
| Outbound | TCP 443 (HTTPS) | Specific IPs (Stripe, Nominatim, OSRM, OAuth, GeoAPI.pt) | External API calls, explicitly listed, not open to the internet |

### database-sg

| Direction | Port / Protocol | Source / Destination | Purpose |
|---|---|---|---|
| Inbound | TCP 22 (SSH) | bastion-sg | Maintenance access |
| Inbound | TCP 27017 | appserver-sg | Application servers reaching MongoDB |
| Inbound | TCP 27017 | database-sg | Replication between PRIMARY and SECONDARY |
| Outbound | TCP 27017 | database-sg | Replica set replication |

---

## Load Balancing and High Availability

Two load balancers run in active-passive mode. The primary serves all normal traffic; the secondary sits in standby. Both run Nginx and Keepalived.

Nginx reverse proxies traffic to the application servers, serves the React frontend as static files, and distributes requests using the `least_conn` algorithm, sending each request to the server with the fewest active connections. The `app_servers.conf` file holds the list of available servers and gets updated dynamically as servers are added or removed, with an immediate Nginx reload.

Keepalived runs VRRP in unicast mode between the two load balancers. The primary holds priority 200 (MASTER) and the secondary holds priority 100 (BACKUP). When the primary fails, Keepalived triggers the `failover.sh` script, which moves the Elastic IP to the secondary via the AWS CLI, making it the new entry point for traffic.

Manual failover via the `kill_lb_primary.sh` script is fully functional and tested. Automatic failover through Keepalived was implemented but showed inconsistent behavior in the AWS environment, so controlled manual failover is the preferred path.

A cron job runs every minute on each load balancer, checking which app servers are still responding. Any that aren't get automatically pulled from the Nginx upstream.

---

## Transport Layer Security

The domain need4rides.xyz has a TLS certificate issued by Let's Encrypt, managed by Certbot with automatic renewal. Nginx is configured to accept TLSv1.2 and TLSv1.3, with an A grade rating. HTTP to HTTPS redirect is enforced. `server_tokens off` keeps the Nginx version out of the response headers.

The HTTP security headers in place are **Content-Security-Policy** (restricts where resources can load from), **Strict-Transport-Security** (forces HTTPS for 1 year, including subdomains), **X-Frame-Options: SAMEORIGIN** (clickjacking protection), and **X-Content-Type-Options: nosniff** (blocks content-type sniffing).

---

## Vulnerability Testing

### External Testing

External tests ran with nmap from a machine outside the VPC, against the public Elastic IP (54.229.43.57). Five scan types were run: a basic scan of the top 1000 ports, service and version detection, a targeted scan of common vulnerable ports (SSH, MongoDB, Redis, MySQL, among others), NSE security scripts (HTTP header analysis and allowed methods), and an SSL/TLS analysis. The results confirmed that only ports 80 and 443 are open, that the SSL/TLS configuration earned an A grade with TLSv1.2 and TLSv1.3, and that every security header is present in the responses.

OWASP ZAP tests (passive scan) found no Critical or High vulnerabilities. The Medium findings relate to the use of `unsafe-inline` in the CSP, which the React frontend needs to function, and to multiple HSTS headers appearing due to multiple location blocks in the Nginx config.

### Internal Testing

Internal tests ran with nmap from each infrastructure instance, simulating the real behavior of each layer. The goal was to confirm that every security group allows only the necessary communication and blocks everything else, including unauthorized internet access. A total of 38 individual checks were run, covering every relevant combination of source, destination, and port. The result was **38/38 PASS**.

From the outside, only ports 80 and 443 are reachable on the public Elastic IP. The bastion can SSH into every other instance but cannot reach any other port or the internet. The load balancers can only talk to the app servers on port 3000 and to the VPC Endpoint on port 443; attempts to reach MongoDB on port 27017 or the internet are blocked. The app servers can only reach MongoDB on port 27017 and the explicitly authorized external APIs on port 443; they cannot reach the load balancers or the general internet. MongoDB nodes can only communicate with each other on port 27017 for replication; all other outbound traffic, including access to the internet, is blocked.

---

## Networking and Security Stack

AWS VPC, AWS EC2 Security Groups, AWS Internet Gateway, AWS NAT Gateway, AWS VPC Endpoint, AWS Elastic IP, AWS CLI, Nginx, Keepalived (VRRP), Let's Encrypt, Certbot, TLSv1.2 and TLSv1.3, SSH with public key authentication, nmap, OWASP ZAP, Bash, Python, Docker, Cron

---

## Application (Frontend, Backend, and Database)

Need4Rides is a ride-hailing platform built with a React frontend (served by the load balancers as static files) and a Node.js/Express backend, exposed through a REST API. Real-time trip tracking runs over Socket.io. The database is MongoDB, set up as a replica set with one PRIMARY and one SECONDARY node for availability and manual recovery in case of failure. Authentication uses JWT, payments are processed through Stripe, and geocoding plus route calculation rely on Nominatim, OSRM, and GeoAPI.pt.

**Stack:** React, Vite, Node.js, Express, Socket.io, MongoDB, Mongoose, Docker, Docker Hub, JWT, bcrypt, Stripe, Nominatim, OSRM, GeoAPI.pt, Locust
