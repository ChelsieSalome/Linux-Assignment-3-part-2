# Linux-Assignment-3-part-2
 
# SALOME CHELSIE LELE WAMBO - AO1372274

# Script Purpose

This project sets up a Bash script to generate a static `index.html` file containing system information. The file is served by an Nginx web server and secured with a firewall using UFW. The system is automated using a systemd service and timer to run daily at 5:00 AM PST.

## Features

- Automates the creation of a system user with a non-login shell.
- Clones necessary scripts and files from a remote repository.
- Generates an HTML page with system information such as:
  - Kernel release
  - Operating system name
  - Current date
  - Number of installed packages
  - System uptime
  - Disk usage (specific to `webgen`)
- Sets up systemd `.service` and `.timer` files for task automation.
- Configures an Nginx server block for serving files.
- Serves the HTML page using an Nginx on port 80.
- Secures the server with UFW by:
  - Allowing SSH and HTTP traffic
  - Enabling SSH rate limiting
- Provides setup instructions for a load balancer serving two servers on DigitalOcean.

## Prerequisites

- An Arch Linux Image uploaded on your DigitalOcean account.
- Basic bash scripting.
- An Arch Linux server.
- Root access or `sudo` privileges.
- Installed packages:
  - `nginx`
  - `ufw`
  - `git`
  Can be installed with `sudo pacman -S nginx ufw git`.  

## Part 1: Repository File Descriptions and Server Locations


### 1. **`setup_script`**
   - **Description**: Bash script designed to automate the setup and configuration of a web server, file server, and associated system services. The script simplifies the process of configuring Nginx, managing systemd services and timers, and setting up essential firewall rules.

   - **Location**: Run from any directory on the server.  

#### Features  

1. **Web and File Server Setup**:
   - Configures an Nginx web server to serve HTML pages and documents.
   - Sets up a file server for serving files with directory browsing enabled.

2. **Systemd Service and Timer Management**:
   - Creates and configures a `generate-index.service` to execute specific tasks.
   - Schedules the `generate-index.service` using `generate-index.timer`.

3. **Firewall Rules**:
   - Configures `ufw` (Uncomplicated Firewall) to allow HTTP and SSH traffic.
   - Limits SSH access to mitigate potential attacks.

4. **System User Management**:
   - Creates a dedicated system user (`webgen`) with a non login shell (**/usr/bin/nologin**)
> **Note**:  
**Why create a system user instead of a regular user or root user**  
- We're essentially running background processes (services, servers), meaning there is little to no interactive tasks performed. A **regular user** ir more appropriate when a person needs to log in, execute commands, or interact with the system. 
- A system user runs with least privileges as opposed to the root user (which grants top-level permissions and thus expose the system to security vulnerabilities)  


5. **Repository Integration**:
   - Clones and integrates required configuration files and scripts from 2 remote repositories.  
   1. "https://github.com/ChelsieSalome/Linux-Assignment-3-part-1.git"  
   2. "https://git.sr.ht/~nathan_climbs/2420-as3-p2-start"  

   - Provides functionality to reset the environment to avoid duplicate files or errors.

6. **Custom Environment options**:
   1. `**[-r]**` : reset option to clean up and remove all files, directories, and configurations created by the script.
   2. `**[-i]**` : Runs the script and execute the commands




### 2. Cloned Files:
The repositories from which the script will clone files, contain the startup files to set up and automate the generation of the `index.html` file. Below is a description of each file and its required location on the server:

 1. `generate_index`
- **Type**: Bash script
- Cloned from "https://git.sr.ht/~nathan_climbs/2420-as3-p2-start"  
- **Description**: script that generates the `index.html` file containing   system information.  
 - **Location**: `/var/lib/webgen/bin/`  

>#### Why this location?
- The `/var/lib/` directory is commonly used for application data files that are dynamically generated or updated.
- Placing the script in `/var/lib/webgen/bin/` ensures it is isolated within the `webgen` user's working environment, preventing interference with other system scripts or applications.
- Keeping the script in the `bin/` directory within the application's structure makes it easy to locate and execute as part of the `webgen` service's logic.
  - **File Explanation**:   
    - **Dynamic System Information:**

        - Automatically updates the HTML file with the system's current kernel version, OS name, date, package count, and public IP address.  

    - **Error Handling:** 
        - Exits immediately on errors to ensure the script does not run in an inconsistent state.
        - Includes detailed error messages for troubleshooting.

    - **Output Location:**  

        - The file is saved in /var/lib/webgen/HTML, which is later used in the server block to congigure nginx.

    - **Usage**:
        - The script `setup_script` automatically makes it executable before running it.


 2. `generate-index.service`
- **Type**: unit (service file)
- **Description**: A systemd service file that runs the `generate_index` script to update the `index.html` file.
- **Location**: `/etc/systemd/system/`

>#### Why this location?
- The `/etc/systemd/system/` directory is reserved for user-defined systemd service and timer files.
- Placing the `generate-index.service` here ensures it is recognized by systemd and can be managed using standard systemd commands (`systemctl enable`, `systemctl start`, etc.).
- This location is accessible to the systemd manager during boot and service initialization.  
- It is recommended to edit files in this location rather than in `/lib/systemd/system/` as this is the default directory that pacman uses to install softwares and updates. Thus files placed are usually overwritten when there are updates and upgrades .  
- **File Explanation**:  
![alt text](image-4.png)  

### `[Unit]` Section
```ini
[Unit]
Description=Generate index.html
Wants=network-online.target
After=network-online.target
```

- **`Description=Generate index.html`**:
  - Provides a brief description of the service for identification in systemd commands.

- **`Wants=network-online.target`**:
  - Indicates that the service prefers the network to be fully initialized before starting but will not fail if the network is unavailable.

- **`After=network-online.target`**:
  - Ensures the service starts only after the `network-online.target` is reached, making it suitable for scripts that depend on network connectivity.

---

#### `[Service]` Section
```ini
[Service]
Type=oneshot
ExecStart=/var/lib/webgen/bin/generate_index
User=webgen
Group=webgen
```

- **`Type=oneshot`**:
  - Specifies that the service runs a single task and then exits. It is not a long-running daemon.

- **`ExecStart=/var/lib/webgen/bin/generate_index`**:
  - The command to execute when the service starts. It runs the `generate_index` script to generate the `index.html` file.

- **`User=webgen`** and **`Group=webgen`**:
  - Runs the service under the `webgen` user and group, ensuring it operates with limited permissions for enhanced security.


#### `[Install]` Section
```ini
[Install]
WantedBy=multi-user.target
```

- **`WantedBy=multi-user.target`**:
  - Indicates that the service should be started in the `multi-user.target`, the typical target for non-graphical multi-user systems.


>**How the Service Works**

1. When started, the service triggers the `generate_index` script.
2. The script collects system information and generates an HTML report (`index.html`).
3. The service runs as a one-time process (`oneshot`) under the `webgen` user for security.




3. `generate-index.timer`
- **Type**: unit (timer file)
- **Description**: A systemd timer file that schedules the `generate-index.service` to run daily at 5:00 AM (PST).
- **Location**: `/etc/systemd/system/`

>#### Why this location?
- Like the service file, the timer file must reside in `/etc/systemd/system/` as this is the preferred location for user-defined unit files.
- It is recommended to edit files in this location rather than in `/lib/systemd/system/` as this is the default directory that pacman uses to install softwares and updates. Thus files placed are usually overwritten when there are updates and upgrades .
- **File Explanation**:  
![alt text](image-3.png)  

#### [Unit] Section  
1. **`[Unit]`**
   - Declares the unit section, which provides metadata about the timer file.

2. **`Description=Timer file for the generate-index.service file`**
   - A short description of the timer.  
#### [Timer] Section  
3. **`[Timer]`**
   - Declares the timer section, which contains the timer's configuration.

4. **`OnCalendar=*-*-* 05:00:00`**
   - Specifies the timer's schedule using the `OnCalendar` directive:
     - `*-*-*` to have the timer run every day. 
     - `05:00:00` specifies the time of day (5:00 AM).
    > Ensure this time is compatible with the timezon on your server. 
    > You can check the timezone with  `timedatectl` and  change it with   `timedatectl set-timezone Region/City` eg: `timedatectl set-timezone America/Vancouver`

5. **`Persistent=true`**
   - Ensures that the timer catches up on missed activations (e.g., if the system was powered off) and runs the associated service as soon as the system restarts.


4. `nginx.conf`
- **Description**: The main Nginx configuration file, modified to include the `sites-enabled` directory for managing server blocks.
- **Location**: `/etc/nginx/`

>#### Why this location?
- `/etc/nginx/` is the default directory for Nginx configuration files on most Linux distributions.
- Placing `nginx.conf` here ensures it is used by the Nginx service during startup.
- Including server blocks via `sites-enabled` in this file maintains modularity, allowing easier management of individual site configurations.  
- **File Explanation**: 

**Global Settings**
 **`user webgen;`**
   - Specifies the system user that the NGINX worker processes run as (`webgen`).

**`worker_processes  1;`**
   - Defines the number of worker processes to handle client requests. In this case, it's set to `1`.  

##### HTTP Block
**`http {`**
   - Starts the `http` block, which contains configurations for handling HTTP traffic.

**`include /etc/nginx/sites-enabled/*;`**
   - Includes all files in the `/etc/nginx/sites-enabled/` directory. This is where additional virtual host configurations are stored.



5. `generate-index.conf`
- **Description**: An Nginx server block configuration file that serves the `index.html` file.
- **Location**: `/etc/nginx/sites-available/`

>#### Why this location?
- The `/etc/nginx/sites-available/` directory is a standard location for storing individual server block configurations in Nginx.
- This structure separates the configuration of individual sites, keeping the main Nginx configuration file cleaner and more maintainable.
- From here, symbolic links are created to `/etc/nginx/sites-enabled/` for active configurations.  
- **File Explanation** :  
![alt text](image.png)
This configuration file defines an NGINX server that:
- Listens on port 80 (IPv4 and IPv6). (**`listen [::];`** & **`listen [::]:80;`**)
- Serves files from `/var/lib/webgen/HTML/`.
- Has a default file `index.html` for directory requests.
- Provides a `/documents` location block with directory listing enabled, mapped to `/var/lib/webgen/documents`.
- Handles root (`/`) requests and serves files if available, returning a 404 error otherwise.



## Part 2: Servers & Load Balancer Creation  

### Setting Up Two Arch Linux Servers and a Load Balancer on DigitalOcean  

This guide walks you through creating two Arch Linux servers on DigitalOcean, configuring SSH access, updating your local SSH configuration, and setting up a load balancer to distribute traffic between the two nealy created servers.  

#### Step 1: Create SSH key-pair  

1. Run `ssh-keygen -t ed25519 -f ~/.ssh/key-name -C "youremail@email.com"` to create the ssh key pair.


2. Choosing a Passphrase  
You can protect the private key using a **passphrase**. A **passphrase** is just a string of characters used to add an additional layer of security to your private key through encryption.  

To make your life easier and avoid the hassle of remembering a passphrase, you can choose not to set one. To do this, simply press **ENTER** twice when prompted for a passphrase. This will create your SSH key without any passphrase protection. Successful completion of the ssh-key should look like:   

3. Verifying your SSH Keys  
Run `cd .ssh` > `ls` to view the files in the .ssh directory.  
> The keys are successfully created if you see the following files in the .ssh directory:  
* **key-name** *(private key)*  
* **key-name.pub** *(public key)*  


2. **Add SSH Key to DigitalOcean**:
   - On your DigitalOcean dashboard, go to **Settings** > **Security**.
   - Click **Add SSH Key** and paste the contents of your public key (`~/.ssh/lb-key.pub`).

#### Step 2: Create Two Arch Linux Droplets

1. **Log in to DigitalOcean**:
   - Access your account at [DigitalOcean](https://www.digitalocean.com/).

2. **Deploy Droplets**:
   - Click **Create** and select **Droplets**.
   - Choose your **Arch Linux** image from the **Custom Images** section.
   - Select the **San Francisco 3 (sfo3)** datacenter region.
   - Choose your desired plan (e.g., Basic with 1 CPU and 1GB RAM).
   - Under **Authentication**, select **SSH Key** and add your public key.
   - Assign the **hostname** as `droplet1` for the first server and `droplet2` for the second.
   - Add the **tag** `web` to both Droplets for easy management.
   - Select the newly created key for authentication when prompted.
   - Click **Create Droplet** to deploy each server.

3. **Note IP Addresses**:
   - After deployment, note the public IP addresses of both Droplets for SSH access and load balancer configuration.

   

#### Step 3: Update Local SSH Configuration

1. **Edit SSH Config File**:
   - On your local machine, edit the SSH configuration file:
   
   - Add the following entries:
     ```Host droplet1
         HostName <doplet1-ip>
         User <username>
         PreferredAuthentications publickey
         IdentityFile ~/.ssh/lb-key
         StrictHostKeyChecking no
         UserKnownHostsFile /dev/null

         Host droplet2
         HostName <doplet2-ip>
         User <username>
         PreferredAuthentications publickey
         IdentityFile ~/.ssh/lb-key
         StrictHostKeyChecking no
         UserKnownHostsFile /dev/null
     ```
   - Replace `<droplet1-ip>`, `<droplet2-ip>`, and `<username>` with the appropriate values.

#### Step 4: Test SSH Access
   - Connect to each Droplet using the configured hostnames:
     ```bash
     ssh droplet1
     ssh droplet2
     ```


#### Step 5: Set Up a Load Balancer

1. **Create the Load Balancer**:
   - On your DigitalOcean dashboard, navigate to **Networking** > **Load Balancers**.
   - Click **Create Load Balancer**.
   - Select the **San Francisco 3 (sfo3)** datacenter region to match your Droplets.
   - Under **Choose Droplets**, select the **web** tag to include both `droplet1` and `droplet2` as backend servers.
   - Configure the **Forwarding Rules**:
     - **Entry Protocol**: HTTP
     - **Entry Port**: 80
     - **Target Protocol**: HTTP
     - **Target Port**: 80
   - Assign the **tag** `web` to the load balancer for consistency.
   - Click **Create Load Balancer** to deploy it.

2. **Configure Health Checks**:
   - In the load balancer settings, ensure health checks are configured to monitor the Droplets' health. This typically involves checking a specific HTTP path to confirm the servers are responsive.  
   ![alt text](image-5.png)


### Notes

- **Consistency**: Ensure both Droplets have identical configurations and content to provide a seamless experience.

- **Scaling**: You can add more Droplets with the `web` tag to the load balancer as needed to handle increased traffic.



### Step 1: Create a System User

1. Create a non-login system user `webgen` with a home directory:

`sudo useradd -r -m -d /var/lib/webgen -s /usr/sbin/nologin webgen`  

> Note:  
Creating a **system user** like `webgen` for this task provides several benefits:
- **Security**: A system user with no login shell and restricted permissions limits potential vulnerabilities. Even if the user is compromised, it cannot directly access other parts of the system.
- **Isolation**: Keeps files and processes for this task separate from other users or services, making management and debugging easier.
- **Minimized Risk**: Running services as root is dangerous because any misconfiguration or compromise could allow full system access.  

2. Ensure the user owns its directory:  

`sudo chown -R webgen:webgen /var/lib/webgen`  
`sudo chmod -R 755 /var/lib/webgen`  

3. Create the required directories:
```bash
mkdir -p /var/lib/webgen/bin
mkdir -p /var/lib/webgen/HTML
```

### Step 2: Clone the Repository

1. Clone this repository into `/var/lib/webgen/bin/`:
```bash
sudo mkdir -p /var/lib/webgen/bin/
sudo -u webgen git clone  https://github.com/ChelsieSalome/Linux-Assignment-3-part-1.git /var/lib/webgen/bin/
```

2. Ensure the following files from the repository are in the correct locations on the server:
    * **`generate_index`** : Script that generates the index.html file. 
        *Already located in `/var/lib/webgen/bin/` after cloning the repository. This script generates the `index.html` file.*  

    * **`generate-index.service`**: Copy this file to `/etc/systemd/system/`:  
        `sudo cp /var/lib/webgen/bin/generate-index.service /etc/systemd/system/`  

    * **`generate-index.timer`**: Copy this file to `/etc/systemd/system/`  
        `sudo cp /var/lib/webgen/bin/generate-index.timer /etc/systemd/system/`  

    - **`nginx.conf`**: This file configures the Nginx server. If there is already an `/etc/nginx/nginx.conf` file on your system, follow these steps:  

    1. **Back up the Existing Configuration**:
     ```sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak```  

    2. **Compare and Merge Changes**:
        Open the existing `/etc/nginx/nginx.conf` and the repository's `nginx.conf` to identify differences. Ensure the following changes are made:  
        
        - Replace the `user` directive (usually near the top of the file) with:  
        ```user webgen;```  
        - Ensure the `http` block includes the following at the end:  
        ```include /etc/nginx/sites-enabled/*;```

    3. **Test the Configuration**:
        After merging the changes, test the Nginx configuration:
        ```sudo nginx -t```  

    4. **Restart Nginx**:  
        Restart the Nginx service to apply changes:  
        ```sudo systemctl restart nginx ```

    5. **If the Backup is Needed**:  
        If something goes wrong, restore the original configuration:  
        ```sudo cp /etc/nginx/nginx.conf.bak /etc/nginx/nginx.conf```  
        ```sudo systemctl restart nginx```  

    * **`generate-index.conf`**: Copy this file to **/etc/nginx/sites-available/**:  
        `sudo cp /var/lib/webgen/bin/generate-index.conf /etc/nginx/sites-available/`
        > **Note**:  
        Using a separate server block file instead of modifying the main nginx.conf file directly offers to following advantages:  
        1. It keeps each site's configuration in its own file which is practical if your server is managing multiple files.  
        2. You can enable or disable individual configurations without affecting the main `nginx.conf`.  
        3. Editing `nginx.conf` directly increases the risk of breaking the entire Nginx setup due to errors in the configuration.  
    

### Step 3: Configure Nginx

1. **Create the Directories (If They Don't Exist)**:  

Nginx does not create `sites-available` and `sites-enabled` by default. Create these directories:  

   ```bash
   sudo mkdir -p /etc/nginx/sites-available
   sudo mkdir -p /etc/nginx/sites-enabled
   ```

2. Enable the Site: Create a symlink in the **sites-enabled directory** to enable the server block:  

`sudo ln -s /etc/nginx/sites-available/generate-index.conf /etc/nginx/sites-enabled/`

3. Test and reload Nginx:  

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Step 4: Enable and Start Systemd Service and Timer

1. Reload systemd:
`sudo systemctl daemon-reload`

2. Enable the service and timer:
```bash
sudo systemctl enable --now generate-index.service
sudo systemctl enable --now generate-index.timer
```
3. To verify that the timer is active and that the service runs successfully, run: 
* `systemctl list-timers` to **Check active timers**.  
* `journalctl -u generate-index.service` to Check service logs.  

### Step 5: Configure UFW

Allow SSH and HTTP traffic and enable ufw:

```bash
sudo pacman -S ufw
sudo ufw allow ssh
sudo ufw allow http
sudo ufw limit ssh
sudo ufw enable
```
>**WARNING !**:  
You mush allow SSH traffic before enabling the ufw service, because if you enable UFW (Uncomplicated Firewall) without allowing SSH traffic, you could inadvertently lock yourself out of your server, especially if you're managing it remotely.  

Check the status:
`sudo ufw status`

### Step 6: Ensure your server is running  
After completing the setup, verify that your server is running and accessible:
1. Open a web browser.  
2. Enter your server's public IP address (e.g., http://<your-server-ip>).  
3. You should see the system information displayed on the webpage as such: 
![alt text](image.png)


# Enhancements to `generate_index` Script  
Possible ways to enhance the `generate_index` script include:  

1. **Adding Error Logs for Key Variables**:  
For critical variables such as `KERNEL_RELEASE`, `DATE`, and `UPTIME`, error messages are logged to standard error (`>&2`) if their retrieval fails.  

2. **Disk Usage Information for the `webgen` User**:
- The script calculates the total disk usage of the `webgen` user directory (`/var/lib/webgen`) using the `du` command:
```DISK_USAGE=$(du -sh /var/lib/webgen 2>/dev/null | awk '{print $1}')```
   - This information is included in the generated `index.html` file.  

3. **Improved Directory Validation**:
   - Before creating the `index.html` file, the script verifies that the target directory (`/var/lib/webgen/HTML`) exists. If it doesn’t, the script logs an error and exits.

4. **HTML Updates**:
   - The generated `index.html` file now includes:
     - Disk usage for the `webgen` user.

# Troubleshooting Tips  
## 1. Nginx Fails to Start or Reload
- Run the following to check the configuration:
```bash
sudo nginx -t
```  
If there are syntax errors, review the changes made to /etc/nginx/nginx.conf or the server block file in /etc/nginx/sites-available/.  

## 2. "generate_index" Script Fails
* Check the logs for the systemd service:  
`journalctl -u generate-index.service`  

* Ensure the script is executable:  
`sudo chmod +x /var/lib/webgen/bin/generate_index`

## 3. Missing Firewall Rules  
* Check the UFW status:
`sudo ufw status verbose`  
It should give you the following output:  
![alt text](image-1.png)

* Ensure you have allowed the necessary ports:  
```bash
sudo ufw allow ssh
sudo ufw allow http
```
## 4. Site not accessible in browser or browser displaying the default Nginx index.html page  
* Verify Nginx is running:
`sudo systemctl status nginx`  

* Clear your browser's cache   
* Use a different browser.  

# Sources
1. https://learning.oreilly.com/library/view/linux-for-system/9781803247946/B18575_02.xhtml#_idParaDest-32  
2. https://wiki.archlinux.org/title/Nginx  
3. https://wiki.archlinux.org/title/Uncomplicated_Firewall  
4. https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units  
5. https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files  
6. https://wiki.archlinux.org/title/Systemd/Timers  





