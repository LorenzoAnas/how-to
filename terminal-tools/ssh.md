To install SSH on Ubuntu, you can install the OpenSSH server package. Here’s a step-by-step guide:

1. **Update the Package List:**  
   Open a terminal and run:
   ```bash
   sudo apt update
   ```

2. **Install OpenSSH Server:**  
   Install the package by running:
   ```bash
   sudo apt install openssh-server
   ```

3. **Check the SSH Service Status:**  
   Verify that the SSH service is active:
   ```bash
   sudo systemctl status ssh
   ```
   If it isn’t running, you can start it with:
   ```bash
   sudo systemctl start ssh
   ```

4. **Configure the Firewall (if applicable):**  
   If you’re using UFW (Uncomplicated Firewall), allow SSH connections:
   ```bash
   sudo ufw allow ssh
   ```
   Then check the firewall status:
   ```bash
   sudo ufw status
   ```

5. **Connect Using SSH:**  
   You can now connect from another machine using:
   ```bash
   ssh username@server_ip_address
   ```
   Replace `username` with your Ubuntu username and `server_ip_address` with the IP address of your Ubuntu system.

This will set up a basic SSH installation. You might consider further securing your SSH setup by modifying the configuration file `/etc/ssh/sshd_config` and restarting the service after making changes:
```bash
sudo systemctl restart ssh
```
