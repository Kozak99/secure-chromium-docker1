# Table of Contents
- [Overview](#overview)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [Configuration](#configuration)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

---

## Overview
This project automates:
1. **System Updates** (using `apt-get update && apt-get upgrade`)
2. **(Optional) Installation** of Docker and Docker Compose
3. **Configuration** of a Docker container running Chromium
4. **Prompts** for environment variables like a custom username and password

The script is intended for Ubuntu servers (20.04+), though it may work on other Debian-based systems with slight modifications.

---

## Features
- **Headless Chromium** container for automation or browsing tasks.
- **Docker**-based isolation for improved security and easy cleanup.
- **Easy Installation** with a straightforward script prompting for necessary inputs.
- **Configurable** environment variables (username, password, and more).

---

## Prerequisites
- Ubuntu 20.04 or later
- **Root or sudo** privileges for installing system packages
- Basic familiarity with the Linux command line
Make the script executable:
chmod +x extnodes.sh
Make the script executable:
bash

chmod +x extnodes.sh
(Optional) Review the script to ensure it meets your security standards.
Usage
Run the script:

bash

./extnodes.sh
The script will:

Check if you followed on Twitter (prompt).
Perform system updates (apt-get update && apt-get upgrade).
Prompt to install Docker and Docker Compose.
Ask for a custom user/password.
Generate a docker-compose.yml in ~/chromium.
Launch the Chromium container via Docker Compose.
Access the service:

By default, the script exposes ports 3050 and 3051.
Open http://<server-IP>:3050 in your browser.
Configuration
docker-compose.yml: Located by default at ~/chromium/docker-compose.yml.
Injects environment variables like CUSTOM_USER and PASSWORD.
If you’d prefer not to store a password in plaintext, move it into a .env file or a secret management service.
Change Timezone: Update the TZ variable in the Compose file if needed.
Security Considerations
Password Storage: Currently, the user-provided password is stored in docker-compose.yml. For production, use a .env file or secrets manager.
Firewall: If you do not want Chromium accessible publicly, restrict inbound traffic to ports 3050 and 3051 (e.g., ufw allow from <your_ip> to any port 3050).
Regular Updates: Keep Docker images updated by running:
bash

docker-compose pull
docker-compose up -d
User Permissions: Consider running Docker as a non-root user (adding your user to the docker group).
Troubleshooting
Permission Denied: If ./extnodes.sh is blocked, ensure chmod +x extnodes.sh was executed.
Docker Command Not Found: Ensure you chose “yes” during installation prompts or install Docker manually.
Port Conflicts: If ports 3050 or 3051 are in use, change them in the docker-compose.yml.
Contributing
Fork this repository.
Create a feature branch (git checkout -b feature/new-feature).
Commit your changes (git commit -m 'Add some feature').
Push to the branch (git push origin feature/new-feature).
Open a pull request to propose your changes.


#!/usr/bin/env bash
###############################################################################
# extnodes.sh – A secure, tested, and performant version
#
# This script:
#   - Confirms user follows you on Twitter
#   - Updates the system (apt-get update && apt-get upgrade)
#   - Installs Docker (optional)
#   - Installs Docker Compose (optional)
#   - Prompts for a custom username/password
#   - Creates a docker-compose.yml for Chromium
#   - Runs the container
#
# Tested on: Ubuntu 20.04/22.04
###############################################################################

# 1) Enable strict modes for better security and error handling.
set -euo pipefail

# 2) Define color codes for pretty output.
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
WHITE='\033[1;37m'
RESET='\033[0m' # Reset color

###############################################################################
# 3) Welcome banner – CRYPTO CONSOLE
###############################################################################
echo -e "${CYAN}=============================="
echo -e "       ${GREEN}CRYPTO CONSOLE${CYAN}        "
echo -e "${CYAN}==============================${RESET}"

echo "Please follow us at: https://x.com/cryptoconsol"
read -p "Have you followed us? (yes/no): " followed

if [[ "${followed}" != "yes" ]]; then
    echo -e "${RED}Please follow us and run the script again.${RESET}"
    exit 1
fi

###############################################################################
# 4) Update and upgrade the system
###############################################################################
echo -e "${GREEN}Updating the system...${RESET}"
sudo apt-get update -y
sudo apt-get upgrade -y

###############################################################################
# 5) Install prerequisites
###############################################################################
echo -e "${GREEN}Installing prerequisites...${RESET}"
sudo apt-get install -y curl ca-certificates gnupg lsb-release

###############################################################################
# 6) Ask if Docker should be installed
###############################################################################
read -p "Do you want to install Docker? (yes/no): " install_docker
if [[ "${install_docker}" == "yes" ]]; then
    echo -e "${GREEN}Installing Docker...${RESET}"
    
    # Official Docker GPG key & repo (for Ubuntu).
    # Reference: https://docs.docker.com/engine/install/ubuntu/
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
      | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
      https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) stable" \
      | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    
    sudo apt-get update -y
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io
    
    echo -e "${GREEN}Checking Docker version...${RESET}"
    docker --version
else
    echo "Skipping Docker installation."
fi

###############################################################################
# 7) Ask if Docker Compose should be installed
###############################################################################
read -p "Do you want to install Docker Compose? (yes/no): " install_compose
if [[ "${install_compose}" == "yes" ]]; then
    echo -e "${GREEN}Installing Docker Compose...${RESET}"

    # Fetch the latest release tag from Docker Compose’s GitHub
    # For production, consider pinning a specific version and verifying checksums
    VER="$(curl -s https://api.github.com/repos/docker/compose/releases/latest \
      | grep tag_name | cut -d '"' -f 4)"
    
    # Download and install Docker Compose
    sudo curl -L \
      "https://github.com/docker/compose/releases/download/${VER}/docker-compose-$(uname -s)-$(uname -m)" \
      -o /usr/local/bin/docker-compose
    
    sudo chmod +x /usr/local/bin/docker-compose
    echo -e "${GREEN}Checking Docker Compose version...${RESET}"
    docker-compose --version
else
    echo "Skipping Docker Compose installation."
fi

###############################################################################
# 8) Prompt for CUSTOM_USER and PASSWORD
###############################################################################
read -p "Enter CUSTOM_USER (username): " custom_user
read -s -p "Enter PASSWORD: " password
echo ""  # Print a newline after password prompt

# NOTE: Storing a password in plain text in docker-compose.yml is not ideal.
#       For production, consider storing it in a .env file or secrets manager.

###############################################################################
# 9) Create the docker-compose.yml for Chromium
###############################################################################
echo -e "${GREEN}Setting up the Chromium directory...${RESET}"
mkdir -p ~/chromium
cd ~/chromium

# Remove any old docker-compose.yml to avoid collisions.
rm -f docker-compose.yml

echo -e "${GREEN}Creating docker-compose.yml...${RESET}"
cat <<EOF > docker-compose.yml
version: '3.3'
services:
  chromium:
    image: lscr.io/linuxserver/chromium:latest
    container_name: chromium_browser
    environment:
      - PUID=1000        # Default user ID (adjust to your local user ID if needed)
      - PGID=1000        # Default group ID (adjust to your local group ID if needed)
      - TZ=Europe/London # Change to your local timezone
      - CUSTOM_USER=${custom_user}
      - PASSWORD=${password}
      - CHROME_CLI=https://www.google.com
    ports:
      - "3050:3000"
      - "3051:3001"
    # security_opt:
    #   - seccomp:unconfined  # Removed for better security
    volumes:
      - /root/chromium/config:/config  # Persists Chromium config to the host
    shm_size: "1gb"
    restart: unless-stopped
EOF

###############################################################################
# 10) Start the Chromium container
###############################################################################
echo -e "${GREEN}Starting Chromium container...${RESET}"
docker-compose up -d

echo -e "${GREEN}Chromium container setup completed!${RESET}"
echo "You can access Chromium via http://<server-ip>:3050 (mapped to container port 3000)."
echo "Also port 3051 -> 3001 if the container uses that."
echo "Done."
