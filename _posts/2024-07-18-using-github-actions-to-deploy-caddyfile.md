---
title: Using GitHub Actions to deploy your Caddyfile
date: 2024-07-18 23:00:00 -0500
categories: [Home Network, Caddy]
tags: [home network, caddy, github actions]
---

# Overview

This guide explains how to set up and deploy a Caddyfile using GitHub Actions with a self-hosted runner.

## Prerequisites
1. **Docker and Docker Compose**: Docker and Docker Compose must be installed on your host machine. Refer to the [Docker website](https://docs.docker.com/engine/install/) for installation on your OS.
2. **Caddyfile**: Chances are if you are viewing this guide, you already have Caddy running. This guide will not show you how to create a Caddyfile. Caddy has excellent documentation on how to do that [here](https://caddyserver.com/docs/caddyfile).

## Requirements
1. **Caddy Container**: Ensure Caddy is installed and running in a Docker container on your server.
2. **GitHub Repository**: Create a repository for storing the Caddyfile.
3. **Self-hosted Runner**: Setup a self-hosted runner on your server where Caddy is running.
4. **GitHub Action**: Create a repository for storing the Caddyfile.

## Steps

### 1. Set Up a Caddy server
1. Create a docker compose file with volumes:

    ```yaml
    version: "3.7"

    services:
    caddy:
        image: caddy:latest
        container_name: caddy_public
        restart: unless-stopped
        cap_add:
        - NET_ADMIN
        ports:
        - "80:80"
        - "443:443"
        - "443:443/udp"
        volumes:
        - /path/on/host/caddy_data:/data
        - /path/on/host/caddy_config:/config
        - /path/on/host/caddy_caddyFile:/etc/caddy # take note of this host location. You will need to define it in GitHub Actions.
    ```
2. Run `docker compose up -d` to start your container.

### 2. Create a GitHub Repository

1. Create a new repository on [GitHub](https://github.com).
2. Create a directory structure as follows:
   ```plaintext
   └── caddy
       └── public
    ```

### 3. Set Up a Self-hosted Runner

> The account you use for GitHub actions must have permission to write to the caddy_file volume on the machine. It also needs permission to restart the caddy docker container. I recommend creating a new account for the github actions service and adding this to the /etc/sudoers file (where git is the name of your account): git ALL=(ALL) NOPASSWD: /bin/cp, /usr/bin/docker restart *
{: .prompt-danger }

1. Go to your repository on GitHub.
2. Click on `Settings` > `Actions` > `Runners`.
3. Click `New self-hosted runner` and follow the instructions to download and configure the runner on the server.
4. Assign a label to the runner, e.g., `t1-public`.

### 4. Create a GitHub Action

1. In your repository, create a directory named `.github/workflows/`
2. Inside `.github/workflows/`, create a file named `deploy-public-caddyfile.yml`
3. Add the following content to the `deploy-public-caddyfile.yml` file and commit the changes:
    > Change the run-on and run commands to match what you defined in the previous steps.
    {: .prompt-warning }

    ```yaml
    name: Deploy Public Caddyfile

    on:
    push:
        branches:
        - main
        paths:
        - 'caddy/public/Caddyfile'

    jobs:
    deploy:
        runs-on: [self-hosted, t1-public]  # Use the label for your specific runner that you defined in step 3
        steps:
        - name: Checkout code
        uses: actions/checkout@v2

        - name: Copy Caddyfile to Docker volume
        run: | # Replace /path/on/host/caddy_caddyFile with the host path in your docker-compose.yml file defined in step 1
            sudo cp ./caddy/public/Caddyfile /path/on/host/caddy_caddyFile/Caddyfile
        shell: bash

        - name: Restart Docker container
        run: |
            sudo docker restart caddy_public
        shell: bash

    ```

### 4. Run the Github action
1. Add your Caddyfile to the caddy/public/ directory and your configuration should be updated.