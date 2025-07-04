
# Installing Ansible on RHEL10

This article explains the process for installing Ansible on a RHEL 10 machine and configuring it to connect to managed nodes.

### Part 1: Installing Ansible on RHEL 10 (Control Node)

Ansible is typically installed on a single "control node" (your RHEL 10 machine) from which you manage your other "managed nodes."

1.  **Enable the EPEL Repository (if necessary):**
    Ansible might not be in the default RHEL repositories. The EPEL (Extra Packages for Enterprise Linux) repository is a community-driven repository that provides high-quality add-on packages for RHEL and compatible distributions.

    First, ensure you have the `epel-release` package. You'll likely need to subscribe your RHEL system to Red Hat Subscription Management for this to work seamlessly.

    ```bash
    sudo dnf install epel-release -y
    ```

    If `epel-release` isn't directly available or you face issues, you might need to manually download the `.rpm` from the Fedora Project EPEL page for your specific RHEL version (e.g., `epel-release-latest-10.noarch.rpm` once RHEL 10 is out).

2.  **Install Ansible:**
    Once the EPEL repository is enabled (or if Ansible happens to be in a default RHEL repo), you can install Ansible using `dnf`:

    ```bash
    sudo dnf install ansible -y
    ```

3.  **Verify Installation:**
    After installation, you can verify that Ansible is installed correctly and check its version:

    ```bash
    ansible --version
    ```

    This command should output information about your Ansible version, Python version, and collection paths.

### Part 2: Configuring User and Privileges to Connect to Managed Nodes

Ansible connects to managed nodes primarily via **SSH**. This means you need an SSH user on the managed nodes with appropriate permissions. Ansible is **agentless**, meaning you don't install any special software on the nodes you're managing, just SSH access and Python (which is usually pre-installed on Linux systems).

#### 1\. SSH User on Managed Nodes

You'll need a user account on each managed node that Ansible can log in as. This user should have sufficient privileges to perform the actions defined in your playbooks.

  * **Best Practice:** Create a dedicated user for Ansible automation (e.g., `ansible_user`) on your managed nodes, rather than using `root` directly for SSH login.

    On **each managed node**:

    ```bash
    sudo useradd -m ansible_user
    sudo passwd ansible_user # Set a strong password
    ```

#### 2\. Authentication Methods (SSH Keys - Recommended)

The most secure and convenient way to authenticate is using SSH keys.

  * **Generate SSH Key Pair (on Control Node):**
    If you don't already have one, generate an SSH key pair on your RHEL 10 control node as the user who will run Ansible (e.g., your regular user, not `root`).

    ```bash
    ssh-keygen -t rsa -b 4096 -C "ansible-key"
    ```

    Press Enter to accept the default location (`~/.ssh/id_rsa`) and passphrase (you can leave it empty for automation, but a passphrase adds security if the key is compromised – Ansible Vault can help manage this).

  * **Copy Public Key to Managed Nodes:**
    Copy the generated public key (`~/.ssh/id_rsa.pub`) to the `~/.ssh/authorized_keys` file of the `ansible_user` on **each managed node**.

    ```bash
    ssh-copy-id ansible_user@managed_node_ip_or_hostname
    ```

    You'll be prompted for the `ansible_user`'s password on the managed node. After this, you should be able to SSH without a password.

    **Alternatively (manual copy):**

    ```bash
    # On Control Node:
    cat ~/.ssh/id_rsa.pub

    # On Managed Node (as ansible_user or with sudo):
    mkdir -p ~/.ssh
    chmod 700 ~/.ssh
    echo "PASTE_YOUR_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
    chmod 600 ~/.ssh/authorized_keys
    chown ansible_user:ansible_user ~/.ssh/authorized_keys # Ensure correct ownership
    ```

#### 3\. Privilege Escalation (Sudo)

Many Ansible tasks require root privileges (e.g., installing packages, modifying system files). You'll configure the `ansible_user` on the managed nodes to use `sudo` without a password (or with a password that Ansible can provide).

  * **Configure `sudo` (on Each Managed Node):**
    Add the `ansible_user` to the `wheel` group (which usually has `sudo` access configured by default on RHEL) or create a specific `sudoers` entry.

    **Option A: Add to `wheel` group (recommended for simplicity on RHEL):**

    ```bash
    sudo usermod -aG wheel ansible_user
    ```

    Then, ensure the `wheel` group is configured for NOPASSWD in `/etc/sudoers`. This is often default on RHEL.

    ```bash
    sudo visudo
    ```

    Look for a line similar to:

    ```
    %wheel  ALL=(ALL)       NOPASSWD: ALL
    ```

    If it's commented out (`#`), uncomment it. If you prefer to require a password for `sudo`, leave it commented or use `%wheel  ALL=(ALL)       ALL`.

    **Option B: Specific `sudoers` entry (more granular):**
    Create a new file in `/etc/sudoers.d/` (this is preferred over editing `/etc/sudoers` directly to avoid syntax errors and simplify updates).

    ```bash
    sudo visudo -f /etc/sudoers.d/ansible_user_nopasswd
    ```

    Add the following line (replace `ansible_user` with your actual username):

    ```
    ansible_user ALL=(ALL) NOPASSWD: ALL
    ```

    Save and exit. Ensure the permissions on this file are `0440`.

#### 4\. Ansible Inventory File

The inventory file (`hosts.ini` or `inventory.yml`) tells Ansible which hosts to manage and how to connect to them.

  * **Create an Inventory File (on Control Node):**
    Create a file (e.g., `~/ansible/inventory.ini`):

    ```ini
    [webservers]
    web1.example.com
    web2.example.com

    [databases]
    db1.example.com ansible_host=192.168.1.100

    [all:vars]
    ansible_user=ansible_user
    ansible_private_key_file=~/.ssh/id_rsa # Path to your private key
    # ansible_ssh_pass="your_ssh_password" # Only if not using keys, or for key passphrase
    # ansible_become=true # Enable sudo
    # ansible_become_method=sudo # Specify sudo (default)
    # ansible_become_user=root # User to become (default is root)
    # ansible_python_interpreter=/usr/bin/python3 # If Python is not at default path
    ```

      * `[webservers]`, `[databases]`: Define groups of hosts.
      * `web1.example.com`: Hostname (Ansible will try to resolve this).
      * `db1.example.com ansible_host=192.168.1.100`: Use an alias and specify the IP.
      * `[all:vars]`: Variables that apply to all hosts in the inventory.
          * `ansible_user`: The remote user to connect as.
          * `ansible_private_key_file`: Path to the private SSH key.
          * `ansible_become=true`: Tells Ansible to use privilege escalation (e.g., `sudo`).
          * `ansible_become_method=sudo`: Specifies `sudo` as the escalation method (default).
          * `ansible_become_user=root`: Specifies that `sudo` should elevate to the `root` user (default).

#### 5\. Test Connectivity

Test that Ansible can connect to your managed nodes:

```bash
ansible -i ~/ansible/inventory.ini all -m ping
```

You should see a "pong" response for each host:

```
web1.example.com | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
db1.example.com | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

If you encounter issues, common culprits are:

  * Firewall on the managed node blocking SSH (port 22).
  * Incorrect SSH key permissions (`.ssh` should be 700, `authorized_keys` 600).
  * Incorrect username or hostname in the inventory.
  * `sudo` configuration not allowing passwordless access or missing `wheel` group.

By following these steps, your RHEL 10 machine will be set up as an Ansible control node, ready to manage your other systems.