# How to deploy your Node.js app to an Ubuntu server in Oracle Cloud for FREE using Docker, PuTTY, DuckDNS, Nginx Proxy Manager and Let's Encrypt - and automatically update it on the server with GitHub Actions

All the tools used are completely free to use. We'll be using:
- Oracle Cloud for creating server instances and for their non-expiring "Always Free" tier
- Docker for containerising and deploying the project
- PuTTY for generating public and private key pairs, and loading into SSH sessions faster
- Nginx Proxy Manager with Let's Encrypt for reverse-proxying and generating SSL certificates
- Ubuntu as the server operating system for its wide usage commercially
- DuckDNS for readable hostnames and for being the DNS challenge when generating our Let's Encrypt certificates
- GitHub Actions for automatically updating the Docker image on the server

Prerequisites:
- npm cli on local machine
- docker cli, Docker desktop on local machine, Docker account
- Node.js project on local machine
- Oracle cloud account
- PuTTY downloaded on local machine
- GitHub account
- git configured, initialised and remote set to existing repository

## Create and configure an Ubuntu Server in Oracle Cloud
### Create and configure a Virtual Cloud Network 

1. After logging in to https://cloud.oracle.com, open the navigation menu and select Networking > Virtual cloud networks

2. Select Create VCN and name it appropriately - this VCN will contain a (collection of) container(s) associated with your project
```
Name: alpha project vcn
```

3. Assign an IPv4 CIDR block to the VCN. Usually, you would have to consider avoiding overlaps when connecting between other VPCs or on-prem networks, but we will assume this is a small project for now.
```
IPv4 CIDR Blocks:
10.0.0.0/16
Specified IP addresses: 10.0.0.0 - 10.0.255.255  (65536 IP addresses)
```

4. Navigate to the new VCN you created and select Subnets > Create Subnet. A subnet is a logical division within the VCN and setting the IPv4 CIDR Block defines the range of IP addresses of the subnet. 

For example:

```
IPv4 CIDR Block:
10.0.1.0/24
Specified IP addresses: 10.0.1.0 - 10.0.1.255 (256 IP addresses)
```

⚠ For the purposes of this tutorial, we will set this as a Public Subnet. However you should usually aim to use Private Subnets whenever possible, and this is critical for sensitive information.

5. From your VCN, navigate to Gateways, then scroll to Internet Gateways > Create Internet Gateway.

```
Name: Internet gateway alpha project
```

6. From the subnet you created, on the Details page, scroll to the bottom and click on "Default Route Table for (VCN name)" > Route Rules > Add Route Rules. Add the following Route Rule:

```
Target Type: Internet Gateway
Destination CIDR Block: 0.0.0.0/0
Target Internet Gateway: Internet gateway alpha project
```

This will make your project accessible via the internet.

7. From your VCN, navigate to Security and select "Default Security List for (VCN name)" > Security rules. Ensure you have all of the following rules and add any of the following rules you do not have:

| Stateless | Source      | IP Protocol | Source Port Range | Destination Port Range | Type and Code | Allows                                                               |
| --------- | ----------- | ----------- | ----------------- | ---------------------- | ------------- | -------------------------------------------------------------------- |
| No        | 0.0.0.0/0   | TCP         | All               | 22                     |               | TCP traffic for ports: 22 SSH Remote Login Protocol                  |
| No        | 0.0.0.0/0   | ICMP        |                   |                        | 3, 4          | ICMP traffic for: 3, 4 Destination Unreachable: Fragmentation Needed |
| No        | 10.0.0.0/16 | ICMP        |                   |                        | 3             | ICMP traffic for: 3 Destination Unreachable                          |
| No        | 0.0.0.0/0   | TCP         | All               | 80                     |               | TCP traffic for ports: 80                                            |
| No        | 0.0.0.0/0   | TCP         | All               | 443                    |               | TCP traffic for ports: 443 HTTPS                                     |
| No        | 0.0.0.0/0   | TCP         | All               | 81                     |               | TCP traffic for ports: 81                                            |
| No        | 0.0.0.0/0   | TCP         | All               | 4000                   |               | TCP traffic for ports: 4000                                          |
- Port 22 is the default port for accessing the server using SSH
- Ports 80 and 443 are default ports for accessing the server using http:// and https://
- Port 81 will be used for Nginx Proxy Manager which will be set up later
- Port 4000 is a custom port for accessing localhost:4000 - can be removed later

### Create the Ubuntu server

1. From the Oracle Cloud website menu, navigate to Compute > Instances > Create Instance

- Name your instance something descriptive of your project - the instance will be used to run your project. 
- Select the Ubuntu image and select the latest standard Canonical Ubuntu version.  

⚠ There may be benefits to choosing a different machine incl. Bare metal however we will keep the shape as the default (AMD) for the purposes of this tutorial.

- Set a VNIC name for your primary VNIC. Ensure the Virtual cloud network and Subnet is set to the existing VCN and subnet you created earlier.

⚠ For some projects, it may be useful to select "Manually assign private IPv4 address" for Private IPv4 address assignment - particularly ones with larger interconnected networks - however we can keep "Automatically assign private IPv4 address" selected assuming this is a small project.
	
- Keep the toggle for "Automatically assign public IPv4 address"
	⚠ This may not be necessary for all projects however as we set our subnet to be public, we will make this project publically accessible.

- In the Add SSH keys section, select "Paste public key" and open the PuTTY Gen application

#### PuTTY Key Generator
1. In PuTTY Key Generator, ensure the type of key to generate is set to RSA and select "Generate". Move your mouse around in the box to generate a random key pai.
2. Copy all the content from inside the box labelled "Public key for pasting into OpenSSH authorized_keys file". Then under Actions > Save the generated key, select Save public key and Save private key, naming your keys something memorable (e.g. related to the project and with the indication that it is a public key or private key).

	⚠ When you try to save your private key, you may get a warning saying "Are you sure you want to save this key without a passphrase to protect it?" - for the purposes of this tutorial, I selected "Yes"

 1. Return to the Add SSH section of the server configuration page.
	
	- In the paste box labelled "SSH public key", paste in the Public Key you copied from PuTTY Key Generator
	⚠ For this tutorial, we will leave "Specify a custom boot volume size" toggled off, however, if you would like to control your boot volume size (e.g. less due to lack of storage left on your account, or more due to a very large project), this can be adjusted. We will also leave "Use in-transit encryption" toggled on and "Encrypt this volume with a key that you manage" toggled off.

2. After hitting Create, you will be taken to the Instance you created. Navigate to the Details section and copy the Public IP address. The username should be "ubuntu" by default, since we created an Ubuntu server instance.
3. Open the PuTTY application.

## PuTTY Configuration

1. In PuTTY Configuration, load the Host Name (e.g. username@123.123.123.123) for your Ubuntu server and set the SSH port to 22. Then in the input field under "Saved Sessions", input a name for this session and his "Save".

| Host Name (or IP address) | Port | Connection type: |
| ------------------------- | ---- | ---------------- |
| `ubuntu@123.123.123.123`  | 22   | SSH              |

| Saved Sessions |
| -------------- |
| alpha-project  |

2. From the Category menu, go to SSH -> Auth -> Credentials and load the private key for the server using Browse

3. In the Category menu, go back to Session, ensure the input field under Saved Sessions still contains the session name you just inputted before hitting "Save" again

4. Select "Open". If a warning pops up, select "Accept" to trust the server.

### Setting up the Dynamic DNS (DDNS) with DuckDNS.org

In order to access the server from a text-based domain (rather than an IP address), we will create a domain in Duck DNS.

1. Go to duckdns.org. After logging in, enter a name for your duckdns subdomain and press "add domain"
2. change the current IP address to the Public IP address of your instance (this can be found by accessing your instance in Oracle Cloud and going to the Details page)

## Configuring the server via SSH

1. In the Ubuntu server terminal, run the following commands:

```bash
sudo apt update
sudo apt upgrade -y
```
### Installing Docker

1. To set up Docker's `apt` repository, run the following commands: ([[How to manually deploy Docker container to Ubuntu Server using Dockerfile/References|References]] 2)

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

2. Install the Docker packages.

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

3. Verify the installation is successful by running the `hello-world` image (optional):

   ```bash
   sudo docker run hello-world
   ```

To remove the `hello-world` container once you have tested docker is working, enter:
```
sudo docker rm -f hello-world
```

4. Verify Docker starts up automatically with the system
```bash
systemctl status docker
```

If Docker does not automatically start up with the system, run the following:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

5. If you're running as a non-root user, you'll need to add yourself to the `docker` group ([[How to manually deploy Docker container to Ubuntu Server using Dockerfile/References|References]] 5) then logout and log back in
```bash
sudo usermod -aG docker ubuntu
exit
```

6. Verify the `docker` command works without `sudo` by running `docker run hello-world` (optional)
```bash
docker run hello-world
```

To remove the `hello-world` container once you have tested docker is working, enter:
```
docker rm -f hello-world
```

7. Create a folder with a recognisable name. This is where the files related to your project will be stored, so they can be used when you try to run them.

## Dockerizing the application

1. In your code editor with your project opened, open the terminal and enter the root directory of the folder you wish to Dockerize. Ensure you are logged in using:

   ```bash
   docker login
   ```

Follow the instructions if you are not already logged in.

2.  In your browser, navigate to [hub.docker.com]() > Repositories > select Create a Repository. For this tutorial, I set the repository name to be `my-backend-project`

3. Create a Dockerfile in the root of the directory you wish to Dockerize ([[How to manually deploy Docker container to Ubuntu Server using Dockerfile/References|References]] 6)

```Dockerfile
FROM node:22-slim

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

  
# Run the backend entry file from the copied repository root

CMD ["node", "index.js"]
```
- Working directory is set to /app as this is convention
- Copy package.json and package-lock.json into the working directory (/app)
- Once we have the package*.json files, we can install the dependencies using `RUN npm ci`, where `ci` stands for "clean install". This means it will only install the exact versions that are specified in the package*.json files.
- After installing the dependencies, copy the source code from the root of the folder you wish to Dockerize and the working directory (/app)
- Run the command to start the project from the entry point (in this case, node index.js)

4. SSH into the server and run these commands to figure out where the user directory is:
```bash
# find where the user directory is
whoami
echo $HOME
ls -la /home
```

4. In the user directory, create a directory with a memorable name - this will be where we store our .env file and other project-related dependencies. 
5. 
```bash
# create a folder for your project in the user directory
mkdir -p /home/ubuntu/backend
cd /home/ubuntu/backend

# create the .env file and add the environmental variables needed to run your project
cat > .env <<'EOF'
DB_URL=postgresql://postgres.argagragawgwag:awgawg@aws-1-eu-central-1.pooler.supabase.com:6543/postgres
SECRET_TOKEN=alpha_secret_token
BCRYPT_SALT_ROUNDS=10
PORT=5000
EOF

# check the .env file exists
chmod 600 /home/ubuntu/backend/.env
```

### Method 1: Dockerizing and deploying using `docker compose` (Recommended)

Since we will be launching Nginx Proxy Manager using `docker compose`, this is the ideal method as it will require slightly less steps later on.

1. In the folder that you just created on your server (that currently contains the .env file), create a docker-compose.yml file:
```bash
cd backend
nano docker-compose.yml
```

2. In the docker-compose.yml file, paste the following contents:
```yml
version: "3.8"
services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile
    image: yasminrei/my-backend-project:latest
    container_name: my-backend-project
    restart: unless-stopped
    ports:
      - "4000:5000"
    env_file:
      - .env
    environment:
      - PORT=5000
```
The host port needs to be the same as the custom port you set up in your route rules when setting up the VCN.
Hit CTRL+S and CTRL+X to save and exit.

3. Navigate back to your project in your code editor, and in the root of the project directory you are containerising, create a `docker-compose.yml` file and paste the following contents:
   
```yml
version: "3.8"
services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile
    image: yasminrei/my-backend-project:latest
    container_name: my-backend-project
    restart: unless-stopped
    ports:
      - "4000:5000"
    env_file:
      - .env
    environment:
      - PORT=5000
```

4. the terminal in your code editor, `cd` into the folder where the `docker-compose.yml` is at the root of and run these commands to build and push an image into your Docker repo:

```bash
docker compose build
docker push yasminrei/my-backend-project:latest
```

5. Go back to the SSH instance of your server and run the following commands:

```bash
docker pull yasminrei/my-backend-project:latest

# update the docker-compose.yml file to use the same tag if changed
cd backend
docker compose pull backend

docker compose up -d --remove-orphans
```

### Method 2: Dockerizing and deploying using `docker build`

1. Navigate back to the terminal in your code editor, `cd` into the folder where the Dockerfile is at the root of and run these commands to build and push an image into your Docker repo:

```bash
docker build -t yasminrei/my-backend-project:latest .
docker push yasminrei/my-backend-project:latest
```

2. Open PuTTY, load your server and SSH into it. Then in the terminal, run the following commands to run the project in a container
```bash
# pull new image
docker pull yasminrei/my-backend-project:latest

# stop & remove existing container (in case it exists)
docker rm -f my-backend-project

# run new container, the environment PORT must be the same as the container port.
docker run -d --name my-backend-project --env-file /home/ubuntu/backend/.env -e PORT=5000 -p 4000:5000 yasminrei/my-backend-project:latest
# -e PORT=5000 -p 4000:5000 -> Host accesses at http://localhost:4000 -> forwarded to container:5000
```

3. Check that the application is running at the specified host port
```bash
curl -v http://localhost:4000
```

4. In your browser, check that the server is running e.g. `coolproject.duckdns.org:4000`. At this moment in time, you will get a "Not secure" error. This is because we have yet to set up the SSL.


## Manually updating the server

Whenever you make changes to the code of your dockerized project, you will need to reflect these changes on the docker image on the server. This is because the image does not automatically update when you push changes. Currently, we don't have GitHub actions (or similar) set up, so we need to do this manually.

You can do this by running the exact same commands used to build, push, pull and run the application for the first time.

## Check the application is running

1. Check that the application is running at the specified host port
```bash
curl -v http://localhost:4000
```

2. In your browser, check that the server is running e.g. `coolproject.duckdns.org:4000`. At this moment in time, you will get a "Not secure" error. This is because we have yet to set up the SSL.

## Adding Nginx Proxy Manager

If you dockerized the application using `docker build`, you will need to follow the steps above to dockerize the application using `docker compose`. Once you have completed this, continue with these instructions.

1. SSH into your terminal and create a new folder for putting all the dependencies related to Nginx Proxy Manager.
```bash
mkdir nginxpm
```

2. Inside the folder, create a `docker-compose.yml` file:
```bash
cd nginxpm
nano docker-compose.yml
```

3. Copy and paste the following into the `docker-compose.yml` file:
```yml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginxproxymanager
    restart: unless-stopped
    environment:
      TZ: "Europe/London"
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    networks:
      - npm_network
  
networks:
  npm_network:
    external: true

```

4. Run the following commands:

```bash
# home/ubuntu/nginxpm
docker network create npm_network
docker compose up -d
```

4. Navigate to the folder containing the `.env` and `docker-compose.yml` file for your project and update the `docker-compose.yml` file
```bash
cd .. # home/ubuntu
cd backend # home/ubuntu/backend
nano docker-compose.yml
```

6. Update the `docker-compose.yml` file with the network information. Since we will be using Nginx Proxy Manager, you can also remove the port declaration.

```yml
version: "3.8"
services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile
    image: yasminrei/my-backend-project:latest
    container_name: my-backend-project
    restart: unless-stopped
    env_file:
      - .env
    environment:
      - PORT=5000
     networks:
       - npm_network

 networks:
   npm_network:
     external: true
```

7. Run the following commands:

```bash
# home/ubuntu/backend
docker compose down
docker compose up -d
```

8. In your browser, navigate to the public IP address for your server instance via port 81. Since duckdns.org is setup, you can use your duckdns.org subdomain to access it.

```
e.g. coolproject.duckdns.org:81
```

You should see the Nginx Proxy Manager admin panel. It is important that you remember your credentials.

9. From the navigation bar, navigate to Certificates and select Add Certificate > Let's Encrypt via DNS

10. In a new browser tab, navigate to duckdns.org and obtain your API key

11. Go back to the Nginx admin panel tab. ([[How to manually deploy Docker container to Ubuntu Server using Dockerfile/References|References]] 5)
- In the Domain Names field, input your duckdns.org subdomain and a wildcard domain name.
- Select DuckDNS as your DNS Provider
- Paste your API key from DuckDns.org
- Hit Save and wait

Example:

| Domain Names             | `coolproject.duckdns.org`, `*.coolproject.duckdns.org` |
| ------------------------ | ------------------------------------------------------ |
| DNS Provider             | DuckDNS                                                |
| Credentials File Content | dns_duckdns_token=fa673c9r-c642-4716-8342-924fafga3165 |
If you get an error, you may need to extend the propagation time to 120 seconds before trying again.

| Propagation Seconds | 120 |
| ------------------- | --- |

12. Go back to the Dashboard > Proxy Hosts > select Add Proxy Host. 

Under "Details":

| Domain Names          | api.coolproject.duckdns.org |
| --------------------- | --------------------------- |
| Scheme                | http                        |
| Forward Hostname / IP | my-backend-project          |
| Forward Port          | 5000                        |
- For the Domain Names, add a subdomain for the project where you would like to access it from. Since my project is an API, I will name it accordingly. However you can set the domain to whatever you like as long as it is based off of your duckdns.org subdomain. You can even set it to the root subdomain, as long as it is not being used already. You can even have multiple subdomains based of your duckdns.org link, which will all take you to the project.
- The Scheme is http since we want to be able to access it using http or https (Nginx Proxy Manager will convert it to https automatically)
- The Forward Hostname needs to be the exact name of the docker container that the project is running as on the server. If you are unsure, SSH into your server and run `docker ps`. The name of the container will be under the "NAMES" column
⚠  If you want to proxy a non-docker service, or if your service is not on the same Docker network as the Nginx Proxy Manager, you'll need to put the IP address here instead. Since we are running the service in the same machine as the Nginx Proxy Manager, that will be localhost.
- The Forward Port must be set to the container port (not the host port)

Under "SSL":

- Select the SSL certificate you just created.
- Toggle Force SSL and HTTP/2 Support on.

Hit Save.

14. Add another Proxy Host, this time for accessing the admin portal. I will set mine as `npm.coolproject.duckdns.org`. 

Under "Details":

| Domian Names          | npm.coolproject.duckdns.org |
| --------------------- | --------------------------- |
| Scheme                | http                        |
| Forward Hostname / IP | nginxproxymanager           |
| Forward Port          | 81                          |
- The forward port is 81, since the default port for the admin panel is port 81.

Under "SSL":

- Select the SSL certificate you just created.
- Toggle Force SSL and HTTP/2 Support on.

15. Check that the SSL and proxy is set up correctly by navigating to the new domain names you have just set up.

## GitHub Actions

Currently, we have to manually redeploy the project on the server by SSH'ing into it, which is time-consuming. We can create a GitHub action to automatically redeploy the project every time we push the changes to the main branch on GitHub.

1. In the root directory of your entire GitHub repository (not the directory you are trying to dockerize - unless they are the same), run this command in your termial to create a GitHub workflow file:

```bash
mkdir .github
cd .github
mkdir workflows
cd workflows
touch ci-deploy.yml
```

2. Inside this file, copy and paste this code:

```yml
name: Build and Deploy

# Workflow is triggered when a push to the main branch is detected
on:
  push:
    branches: [main]
  workflow_dispatch: {}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v6
        with:
          context: ./backend
          file: ./backend/Dockerfile
          push: true
          tags: yasminrei/my-backend-project:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            docker pull yasminrei/my-backend-project:latest
            cd ~/backend || cd backend || exit 0
            docker compose pull backend || true
            docker compose up -d --remove-orphans

```

- If your main branch is not called `main`, adjust this YAML file.
- Adjust the context location to match the directory you have been containerising. Ensure the Dockerfile location for your containerised project is also correctly set.
- Change the instances of the image names to match your Docker Hub repository
⚠ For the purposes of this tutorial, all tags are set to latest rather than being versioned. However it is better & more industry-standard to use versioning when tagging.
- In the deploy script, ensure that the script will cd into the folder you created earlier on your server that contains the .env and docker-compose.yml files associated with your project, before running `docker compose` scripts

3. In your browser, navigate to hub.docker.com and click on your profile image > Account settings. Then from the left sidebar menu, click on Settings > Personal access tokens.

4. Select Generate new token and set the Scope to Read & Write. Add a descriptive name so that you can remember what exactly the token is for. Copy the token.

5. In your code editor's terminal, run the command, pasting the token you just copied inside speech marks after `gh secret set DOCKERHUB_TOKEN --body`. The text after the `-R` needs to be {github-username-hosting-the-github-repo}/{github-repository-name}
```bash
gh secret set DOCKERHUB_TOKEN --body "token-you-just-copied-from-docker-here" -R GitHubUser12456/cool-github-repo
```

6. Run the following commands, adjusted based on your actual details (including replacing the github)

```bash
gh secret set DOCKERHUB_USERNAME --body "yasminrei" -R GitHubUser12456/cool-github-repo
gh secret set SSH_HOST --body "123.123.123.123" -R GitHubUser12456/cool-github-repo
gh secret set SSH_USER --body "ubuntu" -R GitHubUser12456/cool-github-repo
```

7. Open the PuTTY Gen application and load your private key for your server.

8. Select Conversions > Export OpenSSH key. Write a file name that does not override your original private key file
	⚠ You may be prompted if you are sure you want to sav this key without a passphrase to protect it. For the purposes of this tutorial, I selected yes.

9. Navigate to the new OpenSSH key file that you just saved and copy the full file path.

10. In your terminal, run:
```bash
gh secret set SSH_PRIVATE_KEY --body (Get-Content -Raw "C:\Users\yasminrei\Downloads\my-private-key") -R GitHubUser12456/cool-github-repo
```

11. Commit and push the workflow yaml file to your repository. In your browser, navigate to your GitHub repository and under the Actions section, check that it is working.

