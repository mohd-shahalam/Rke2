#!/bin/bash
 
# --- Configuration ---
DEBUG_LEVEL=6 # Increase for more verbose RKE2 logging (e.g., 4, 6)
STARTUP_DELAY=5 # Seconds to wait at strategic points (for testing only)
# --- End Configuration ---
 
# Function to log messages
log() {
    echo -e "\e[32m$1\e[0m"
}
 
# Function to log errors
error() {
    echo -e "\e[31m$1\e[0m" >&2
}
 
# Disable swap
log "Disabling swap..."
if sudo swapoff -a && sudo sed -i '/swap/d' /etc/fstab; then
    log "Swap disabled successfully."
else
    error "Failed to disable swap."
    exit 1
fi
 
# Ensure /usr/local/bin is in PATH
if ! echo "$PATH" | grep -q "/usr/local/bin"; then
    echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
    source ~/.bashrc
    log "/usr/local/bin added to PATH."
fi
 
# Function to install Git based on the Linux distribution
install_git() {
    log "Detecting Linux distribution and installing Git..."
    if [ -f /etc/os-release ]; then
        . /etc/os-release
        case "$ID" in
            ubuntu)
                log "Detected Ubuntu. Installing Git..."
                if sudo apt-get update && sudo apt-get install -y git; then
                    log "Git installed successfully on Ubuntu."
                else
                    error "Failed to install Git on Ubuntu."
                    exit 1
                fi
                ;;
            centos|rhel|fedora)
                log "Detected CentOS/RHEL/Fedora. Installing Git..."
                if sudo yum install -y git; then
                    log "Git installed successfully on CentOS/RHEL/Fedora."
                else
                    error "Failed to install Git on CentOS/RHEL/Fedora."
                    exit 1
                fi
                ;;
            *)
                error "Unsupported Linux distribution: $ID"
                exit 1
                ;;
        esac
    else
        error "Unable to detect Linux distribution."
        exit 1
    fi
}
 
# Call the install_git function
install_git
 
# Fetch the public IP address of the master node
log "Fetching the public IP address of the master node..."
MASTER_NODE_PUBLIC_IP=$(curl -s https://api.ipify.org)
if [ -z "$MASTER_NODE_PUBLIC_IP" ]; then
    error "Failed to fetch the public IP address."
    exit 1
fi
log "Public IP address fetched: $MASTER_NODE_PUBLIC_IP"
 
# Create the /etc/rancher and rke2 directories if they don't exist
log "Creating the /etc/rancher/rke2 and rke2 directories if they don't exist..."
if sudo mkdir -p /etc/rancher/rke2 && mkdir -p rke2; then
    log "Directories created."
else
    error "Failed to create directories."
    exit 1
fi
 
# Create and configure the config.yaml file with increased debug level
log "Creating and configuring the config.yaml file with debug level $DEBUG_LEVEL..."
cat <<EOL | sudo tee /etc/rancher/rke2/config.yaml > /dev/null
write-kubeconfig-mode: "0644"
tls-san:
    - "$MASTER_NODE_PUBLIC_IP"
debug: true
v: $DEBUG_LEVEL
EOL
if [ $? -eq 0 ]; then
    log "config.yaml file created and configured with debug level $DEBUG_LEVEL."
else
    error "Failed to create config.yaml file."
    exit 1
fi
 
# Install RKE2
log "Installing RKE2..."
if curl -sSfL https://get.rke2.io | sh -; then
    log "RKE2 installation completed."
else
    error "Failed to install RKE2."
    exit 1
fi
 
# Enable and start nm-cloud-setup.service
log "Enabling and starting nm-cloud-setup.service..."
if sudo systemctl enable nm-cloud-setup.service && sudo systemctl start nm-cloud-setup.service; then
    log "nm-cloud-setup.service enabled and started successfully."
else
    error "Failed to enable or start nm-cloud-setup.service."
    exit 1
fi
 
# Add a delay after nm-cloud-setup (for testing)
if [ "$STARTUP_DELAY" -gt 0 ]; then
    log "Waiting $STARTUP_DELAY seconds after nm-cloud-setup (for testing)..."
    sleep "$STARTUP_DELAY"
fi
 
# Modify rke2-server.service to comment out the problematic line
log "Modifying rke2-server.service to comment out the problematic line..."
if sudo sed -i 's|^ExecStartPre=/bin/sh -xc '\''! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service'\''|#&|' /usr/lib/systemd/system/rke2-server.service; then
    log "Modified rke2-server.service successfully."
else
    error "Failed to modify rke2-server.service."
    exit 1
fi
 
# Reload systemd daemon
log "Reloading systemd daemon..."
if sudo systemctl daemon-reload; then
    log "Systemd daemon reloaded successfully."
else
    error "Failed to reload systemd daemon."
    exit 1
fi
 
# Enable the rke2-server.service
log "Enabling the rke2-server.service..."
if sudo systemctl enable rke2-server.service; then
    log "rke2-server.service enabled."
else
    error "Failed to enable rke2-server.service."
    exit 1
fi
 
# Add a delay before starting rke2-server (for testing)
if [ "$STARTUP_DELAY" -gt 0 ]; then
    log "Waiting $STARTUP_DELAY seconds before starting rke2-server (for testing)..."
    sleep "$STARTUP_DELAY"
fi
 
# Start the rke2-server.service
log "Starting the rke2-server.service..."
if sudo systemctl start rke2-server.service; then
    log "rke2-server.service started."
else
    error "Failed to start rke2-server.service."
    exit 1
fi
 
# Create the .kube directory in the home directory if it doesn't exist
if mkdir -p $HOME/.kube; then
    log ".kube directory created or already exists."
else
    error "Failed to create .kube directory."
    exit 1
fi
 
# Copy kubectl to /usr/local/bin
if sudo cp /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/; then
    log "kubectl copied to /usr/local/bin."
else
    error "Failed to copy kubectl to /usr/local/bin."
    exit 1
fi
 
# Make kubectl executable
if sudo chmod +x /usr/local/bin/kubectl; then
    log "kubectl is now executable."
else
    error "Failed to make kubectl executable."
    exit 1
fi
 
# Copy the RKE2 kubeconfig to the .kube directory
if sudo cp /etc/rancher/rke2/rke2.yaml $HOME/.kube/config; then
    log "Kubeconfig copied to $HOME/.kube/config."
else
    error "Failed to copy kubeconfig."
    exit 1
fi
 
# Change ownership of the kubeconfig file
if sudo chown $(id -u):$(id -g) $HOME/.kube/config; then
    log "Ownership of kubeconfig file changed for $HOME/.kube/config."
else
    error "Failed to change ownership of kubeconfig file."
    exit 1
fi
 
# Set the KUBECONFIG environment variable
export KUBECONFIG=$HOME/.kube/config
log "KUBECONFIG environment variable set to $HOME/.kube/config."
 
 
# Generate the token
log "Generating the node token..."
NODE_TOKEN=$(sudo cat /var/lib/rancher/rke2/server/node-token)
if [ -z "$NODE_TOKEN" ]; then
    error "Failed to generate the node token."
    exit 1
fi
log "Node token generated: $NODE_TOKEN"
 
# Display the server certificate
log "Displaying the server certificate..."
SERVER_CERT=$(sudo cat /var/lib/rancher/rke2/server/tls/server-ca.crt)
if [ -z "$SERVER_CERT" ]; then
    error "Failed to display the server certificate."
    exit 1
fi
log "Server certificate:\n$SERVER_CERT"
log "All steps completed successfully."
