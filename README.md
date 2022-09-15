# Setup reverse proxy / firewall for self hosting services available through FQDN

## Example outlines self hosting a Gitlab instance with private docker container registry

### Links

- Cloud Provider: **[Azure](https://portal.azure.com/)**
- Password Manager: **[Bitwarden](https://vault.bitwarden.com/)**
- Self-hosted VM manager: **[Proxmox](https://www.proxmox.com/en/)**
- Gitlab:
  - **[About](https://about.gitlab.com/)**
  - **[Installation Docs](https://about.gitlab.com/install/)**

### Steps Taken:

1. Spin up an Ubuntu machine that has a static IP

   - <code>sudo apt udpate</code>
   - <code> sudo apt upgrade</code>

2. Install **[Wireguard](https://www.wireguard.com/)** - _[Instructions for Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04)_

   - for step 2(a) make sure you're not interupting the server's current eth0 ip range. i.e. if the Ubuntu machine has a CIDR block 10.0.0.0/24 10.0.0.0 - 10.0.0.255 is taken. You can use <code>ip addr</code> to see what is currently being used for eth0
   - no need for step 2(b)
   - no need for step 4
   - stop after step 6 for now
   - make sure wireshark is running and <code>ip addr</code> shows an eth0 and wg0 network interface
   - make sure that whatever firewall this Ubuntu machine is behind also allows connection through the UDP wireguard port

3. Install **[Firewalld](https://firewalld.org/)** - _[Instructions for Ubuntu](https://computingforgeeks.com/install-and-use-firewalld-on-ubuntu/)_
   - After Install: (don't restart the firewalld server during this as your changes aren't saved until the end. you can use <code>--permanent</code> on each command to save as you go)
     - <code>sudo firewall-cmd --new-zone=external</code>
     - <code>sudo firewall-cmd --zone=external --add-service={http,https}</code>
     - <code>sudo firewall-cmd --zone=external --add-port=<u><em>setup-wireshark-port</em></u>/udp</code>
     - <code>sudo firewall-cmd --zone=external --add-interface=eth0</code>
     - <code>sudo firewall-cmd --get-zone-of-interface=eth0</code> should return <code>external</code>
     - external firewall is configured. Now the internal
     - <code>sudo firewall-cmd --new-zone=internal</code>
     - <code>sudo firewall-cmd --zone=internal --add-interface=wg0</code>
     - <code>sudo firewall-cmd --get-zone-of-interface=wg0</code> should return <code>internal</code>
     - internal firewall is configured
     - <code>sudo firewall-cmd --zone=external --list-all</code> shows external interface current config
     - <code>sudo firewall-cmd --zone=internal --list-all</code> shows internal interface current config
     - if all looks good and <code>--permanent</code> wasn't used throughout run the next command to save current changes
     - <code>sudo firewall-cmd --runtime-to-permanent</code>

What is setup now is a firewall with two sepearte configurations. One for the external eth0 network interface and one for the internal wg0 network interface. Anybody navigating to the server's static IP will first have to make it through the firwall external zone. I setup a FQDN that uses subdomains that will end up being broght to different wg0 ips and ports based on the Nginx configuration file for that FQDN. If you don't have a FQDN you might have to open more ports as you won't be able to differentiate the incomming http requests as being different like you can with a subdomain address.

We now have a firewall setup that allows us to masquerade and port forward external ips hitting our internet-connected eth0 network interface and forward that into our interal wireguard wg0 netowrk interface if required. Becuase we're going to be setting up an Nginx reverse proxy listening on eth0 we don't have to do this specifically but this setup allows the option.

4.  Setup VM or machine that will run Gitlab and Gitlab-Runner

    1.  Spin up Ubuntu Server installation
        - <code>sudo apt udpate</code>
        - <code> sudo apt upgrade</code>
    2.  Setup a Wireguard Peer - _[continuation](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04#step-7-configuring-a-wireguard-peer)_

        - Example wg0.conf file below

        <code>

            [Interface]
            PrivateKey = gitlab_vm_private_key
            Address = 10.1.0.10/24
            MTU = 1420

            [Peer]
            PublicKey = ubuntu_VPS_server_public_key
            AllowedIPs = 10.1.0.0/24
            Endpoint = ubuntu_public_ip_address:wireguard_port
            PersistentKeepalive = 25

        </code>

    3.  Connect server to peer
        - After the .conf file is setup we need to add the peer to the Ubuntu server
        - <code>sudo wg set wg0 peer <u><em>gitlab_vm_public_key</em></u> allowed-ips 10.1.0.10</code>
        - After this you should be able to connect to and ping the Ubuntu server from the ip address setup during the wireguard installation for the Ubuntu server
    4.  Install docker

        - <code>curl -fsSL https://get.docker.com -o get-docker.sh</code>
        - <code>sudo sh get-docker.sh</code>
        - <code>rm get-docker.sh</code>
        - Docker and Docker Compose are now installed. Verify by running <code>sudo docker --version</code> and <code>sudo docker compose version</code>
        - <code>sudo groupadd docker</code>
        - <code>sudo usermod -aG docker $USER</code>
        - <code>newgrp docker</code>
        - <code>mkdir ~/gitlab && mkdir ~/gitlab-runner</code>
        - <code>sudo mkdir /etc/gitlab && mkdir /etc/gitlab/config && mkdir /etc/gitlab/data && mkdir /etc/gitlab/logs</code>
        - <code>nano ~/gitlab/docker-compose.yml</code> and copy docker-compose.gitlab.yml file
        - <code>nano ~/gitlab-runner/docker-compose.yml</code> and copy docker-compose.gitlab-runner.yml file
        - <code>sudo nano /etc/systemd/system/gitlab.service</code>

        <code>

            [Unit]
            Description=Gitlab
            Requires=docker.service
            After=docker.service
            [Service]
            Restart=always
            User=username
            Group=docker
            WorkingDirectory=/home/username/gitlab
            # Shutdown container (if running) when unit is stopped
            ExecStartPre=/usr/bin/docker compose -f docker-compose.yml down -v
            # Start container when unit is started
            ExecStart=/usr/bin/docker compose -f docker-compose.yml up
            # Stop container when unit is stopped
            ExecStop=/usr/bin/docker compose -f docker-compose.yml down -v
            [Install]
            WantedBy=multi-user.target

        </code>

        - <code>sudo nano /etc/systemd/system/gitlab-runner.service</code>

        <code>

            [Unit]
            Description=Gitlab Runner
            Requires=docker.service
            After=docker.service
            [Service]
            Restart=always
            User=username
            Group=docker
            WorkingDirectory=/home/username/gitlab-runner
            # Shutdown container (if running) when unit is stopped
            ExecStartPre=/usr/bin/docker compose -f docker-compose.yml down -v
            # Start container when unit is started
            ExecStart=/usr/bin/docker compose -f docker-compose.yml up
            # Stop container when unit is stopped
            ExecStop=/usr/bin/docker compose -f docker-compose.yml down -v
            [Install]
            WantedBy=multi-user.target

        </code>

        - We're not going to start these quite yet

5.  Setup FQDN

    - Get a domain name and set the DNS records to allow a subdomain of gitlab.domain.com and registry.gitlab.domain.com pointing to the static ip of the Ubuntu VPS server

6.  Setup NGINX Reverse Proxy

    - <code>sudo apt install -y nginx</code>
    - NGINX automatically installs a systemd service for auto startup on boot
    - <code>sudo nano /etc/nginx/sites-available/domain.com</code>

          map $http_upgrade $connection_upgrade {
          default upgrade;
            '' close;
          }

          server {
                server_name gitlab.domain.com;

                location / {
                    proxy_pass http://gitlab_wg0_ip_address:9080/;
                    proxy_read_timeout 3600s;
                    proxy_http_version 1.1;
                    proxy_set_header Upgrade $http_upgrade;
                    proxy_set_header Connection $connection_upgrade;
                }

                access_log /etc/nginx/sites-available/gitlab.access.log;
                error_log /etc/nginx/sites-available/gitlab.error.log info;

          }

          server {
                  listen 5000 ssl;
                  server_name registry.gitlab.domain.com;

                  client_max_body_size 0;

                  location / {
                          proxy_pass http://gitlab_wg0_ip_address:5000/;
                          proxy_read_timeout 3600s;
                          proxy_http_version 1.1;
                          proxy_set_header Upgrade $http_upgrade;
                          proxy_set_header Connection $connection_upgrade;
                  }

                  access_log /etc/nginx/sites-available/registry.access.log;
                  error_log /etc/nginx/sites-available/registry.error.log info;
          }

    - <code>sudo ln -s /etc/nginx/sites-available/domain.com /etc/nginx/sites-enable/domain.com</code>
      - these paths HAVE to start from the / root path or the link will not work correctly
      - Now we need to generate the SSL certs for the FQDN using certbot
    - <code>sudo apt-cache policy certbot</code>
    - <code>sudo apt-get -y install certbot</code>
    - <code>sudo certbot --nginx</code>
    - This will walk you through an interactive shell for selecting and auto redirecting existing server configurations to use SSL
    - Certbot should have a service that automatically checks for ssl certs that need to be renewed
    - <code>nginx -t</code> will make sure the current NGINX configuration are correct
    - <code>nginx -s reload</code> will soft restart the nginx service with the changed configurations

At this point we have a FQDN pointing to our static server with subdomains = [gitlab.domain.com, registry.gitlab.domain.com]. The HTTP and HTTPS services are already setup for the external firewall and will get allowed into the network at which point the NGINX proxy will recognize the domain and proxy it to the gitlab Ubuntu server thats connected through the Wireguard vpn tunnel. The only issue is that we're currently not running the gitlab or gitlab-runner containers. So let's do that now.

7. Start Gitlab
   - <code>sudo systemctl enable gitlab</code>
   - <code>sudo systemctl enable gitlab-runner</code>
   - <code>sudo systemctl start gitlab</code>
   - <code>sudo systemctl start gitlab-runner</code>
   - You can monitor the status of the processes by running <code>journalctl -xefu gitlab</code>
   - First spin this will take ~10 minutes to pull down the docker container and setup the instance
   - while it's spinning up run <code>sudo cat /etc/gitlab/config/initial_root_password</code> to get the password you will need for initial login

Once this is done spinning up you should be able to access the login page by navigating to https://gitlab.domain.com

8.  Configure Gitlab

    1.  Login the username: root password: _copied_inital_password_
    2.  Menu > Admin > Settings > General
        - expand Visibility and Access Limits
          - change Enable Git access protocols to "Only HTTP(S)"
        - expand Sign-up restrictions
          - uncheck Sign-up enabled
    3.  Menu > Admin > Settings > CICD
        - expand Continuous Integration and Development
          - uncheck Default to Auto DevOps pipeline for all projects
    4.  Menu > Admin > Overview
        - New User & create a user for yourself
    5.  Menu > Groups > Create Group
    6.  Menu > Projects > Create Project > Create from template & select NodeJS Express
    7.  Menu > Projects > NEW_PROJECT > CI/CD > Editor

            build-job:
            script:
                - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER $CI_REGISTRY --password-stdin
                - docker build -t $CI_REGISTRY_IMAGE .
                - docker push $CI_REGISTRY_IMAGE

        - commit changes and the pipeline should run and pass with the container showing up in the Package & Registries > Container Registry page

Hopefully everthing is working lol

## Remote Git Commands

- Menu > Group > Settings > Access Tokens & create an access token

- Make sure Git Credentials Manager is installed. If on Windows during the Git installation make sure you select GCM as an option during install steps.

- <code>git clone gitlab.domain.com/group_name/repo_name</code> and it will prompt you for username/token
- this will be saved for following git commands

## Local Network Git Commands

- <code>git clone local_ip:9080/group_name/repo_name</code> and it will prmpt you for username/token

## Remote Docker Commands

- <code>docker login -u username registry.gitlab.domain.com</code> & enter password
- <code></code>

## Local Network Docker Commands

- Need to update docker's insecure registry settings
- on Windows it's in C:\Users\username\\.docker\daemon.json

       "insecure-registries": ["local_ip:5000"]

- <code>docker login -u username local_ip:5000</code>
- <code>docker pull local_ip:5000/group_name/repo_name</code>
