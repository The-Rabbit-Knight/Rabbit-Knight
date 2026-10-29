---
title: "Deploying Peppermint Helpdesk on Arch Linux: A Lightweight Ticketing System for Your IT Lab"
date: 2025-07-03
medium_url: https://medium.com/@rabbit_knight/deploying-peppermint-helpdesk-on-arch-linux-a-lightweight-ticketing-system-for-your-it-lab-8f0d4c9a2cd0
---

> **Original Published Article on Medium:** [Deploying Peppermint Helpdesk on Arch Linux: A Lightweight Ticketing System for Your IT Lab](https://medium.com/@rabbit_knight/deploying-peppermint-helpdesk-on-arch-linux-a-lightweight-ticketing-system-for-your-it-lab-8f0d4c9a2cd0)

### Deploying Peppermint Helpdesk on Arch Linux: A Lightweight Ticketing System for Your IT Lab

![Article Figure](https://cdn-images-1.medium.com/max/800/1*5MjrTxP5k5sCcneEtZkvqA.jpeg)

If you’re preparing for an interview or building a proof of concept, setting up a helpdesk ticketing system using your home lab can help simulate real-world IT workflows. Most tutorials focus on enterprise platforms like ServiceNow, but these tools can be overkill, especially for beginners or solo learners.

That’s where Peppermint Helpdesk comes in. It’s a lightweight, open-source, and Docker-friendly solution perfect for smaller-scale environments. It may not have advanced automation, but it provides all the core features you’d expect in a simple ticketing system.

In this guide, I’ll walk you through deploying Peppermint on an Arch Linux-based system (I used CachyOS) using Docker Compose.

#### Why Choose Peppermint?

- Lightweight & Open-Source
- Simple setup — Easily deployable via Docker
- Core helpdesk features — User management, ticket creation, workflow
- Clean, minimal interface — Peppermint’s interface is simple, but it does the job

#### My Setup

- OS: CachyOS (Arch-based Linux distro)
- Docker & Docker Compose installed (see below)
- A web browser to access the web interface

#### Step-by-Step Deployment

#### Step 1: Update the System

Always start with an updated system. Since CachyOS follows a rolling release model, updating also ensures compatibility with the latest Docker packages.

```
sudo pacman -Syu
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*2PWwsaiYLFFZsb7hf5BL1Q.png)

#### Step 2: Visit the Peppermint Project

Before setting anything up, it’s a good idea to check out the project itself. You can visit Peppermint’s official website here: https://peppermint.sh . Take a few minutes to explore what it offers and how it works. Scroll down the homepage until you see the documentation section. That part has helpful links — including how to deploy it using Docker.

#### Step 3: Install Docker and Docker Compose

You’ve probably heard people describe Docker as a way to package apps into neat little containers so they run the same way anywhere. That’s pretty accurate — but to make that happen, you first need to install Docker itself. Docker is the engine that runs those containers.

Now, when a project (like Peppermint) uses multiple containers — say, one for the app and another for the database — you’ll want an easier way to manage them together and orchestrate their launch sequence. That’s where Docker Compose comes in. It’s like a foreman building a finished solution from multiple containers. It reads a special YAML file where you define how different containers work together. You can start a whole group of containers with a single command.

The two commands below will install both Docker and Docker Compose, so you’re ready to deploy Peppermint.

```
sudo pacman -S docker
sudo systemctl 
enable
 --now docker
```

In Arch, Docker Compose is provided via a separate package. Install it via:

```
sudo pacman -S docker-compose
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*NtrGRkxxR452eveDPiHMCQ.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*T5lKjndwu_TFOxilhXO8MQ.png)

#### Step 4: Create Your Project Directory

```
mkdir
 ~/peppermint
cd
 ~/peppermint
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*oPowix-3RnmZ6SG4gd-Spw.png)

#### Step 5: Create a docker-compose.yml File

When you run Docker Compose, it reads this configuration file docker-compose.yml . Use a text editor like vim or nano or any you like. Then paste in the latest configuration from the Peppermint GitHub documentation

![Article Figure](https://cdn-images-1.medium.com/max/800/1*1yVOtSAzAuD5BpDcUrREBA.png)

#### Step 6: Start the Container Stack

```
sudo docker-compose up -d
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*koTOL1pidOk_pspTHZUPdg.png)

Once deployed, open your browser and go to: http://localhost:3000

Log in with default credentials (check the official documentation).

Tip: Create a new admin account and delete the default one for better security, even in a lab!

![Article Figure](https://cdn-images-1.medium.com/max/800/1*N041WsnlmRJbuxKrCOcl0g.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*jqc5lyOHx67Mxyfw83u3sQ.png)

#### Creating and Managing Tickets

Let’s walk through a basic example of how to create and resolve an issue in Peppermint.

Scenario : Suspicious login attempt detected.

1. Log in as a Client/User: Access the platform using a client or end-user account. This simulates how an employee or customer would report a problem.
2. Report a New Issue: Click “New Issue”.
3. Provide Issue Details: Title, description, fill out any other required fields

![Article Figure](https://cdn-images-1.medium.com/max/800/1*g5mNClQSEwFMcjaMb6te9g.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*C6Xw1GzsU0HYGRsMcYCSOA.png)

4. Access the Ticket Queue: Log in as a security analyst (or your mock analyst account) and go to the main ticket queue or dashboard.

5. Assign the Ticket: Locate the newly created ticket and formally assign it to yourself (or your designated analyst account) to indicate you are taking ownership.

6. Document Investigation Steps: Add notes to the ticket detailing every step of your investigation

7. Close the Ticket: Once the investigation is complete and the issue is resolved (or mitigated), change the ticket’s status to “Closed.”

![Article Figure](https://cdn-images-1.medium.com/max/800/1*-8uHnLBDVwZ07RykZu2JSQ.png)

You can use this method to simulate multiple types of incidents, like phishing emails, malware alerts, or privilege misuse.

#### Shut Down & Clean Up

To stop all containers launched for the Peppermint application stack, execute the following CLI command:

```
sudo docker-compose down
```

#### Final Thoughts

While Peppermint isn’t built for enterprise environments, it’s a great tool for learning and simulating helpdesk workflows. Whether you’re studying for IT certifications, practicing incident documentation, or — like me — just getting used to working with tickets, it’s a great way to build routine and confidence.
