# Tutorial: Setting Up and Managing Virtual IPs on a VPS with LXC Containers and NAT

## Introduction

This tutorial will guide you through setting up and managing virtual IPs on a VPS using LXC containers, NAT, and routing with `iptables`. You'll also learn how to automate these tasks with a Discord bot that uses both slash commands and prefix commands.

## Prerequisites

1. **VPS with a static IP address**: Ensure your VPS provider allows the creation of additional virtual IPs.
2. **Ubuntu-based server**: This tutorial uses Ubuntu for configuration.
3. **Python 3.8+**: Ensure Python is installed on your VPS.
4. **Git**: For managing your project repository.
5. **Discord Bot Token**: Create a bot and obtain the token from the Discord Developer Portal.

## Dependencies

Install the necessary dependencies on your VPS:

```bash
sudo apt-get update
sudo apt-get install -y lxc lxc-templates iptables python3-pip git
pip install -U discord.py
```

## Step 1: Setting Up LXC Containers and Network Configuration

### 1.1 Creating the `manage_users.py` Script

Create a Python script `manage_users.py` to manage LXC containers and network configuration.

```python
import subprocess

def run_command(command):
    """Run a shell command."""
    result = subprocess.run(command, shell=True, capture_output=True, text=True)
    if result.returncode != 0:
        print(f"Error: {result.stderr}")
        return None
    return result.stdout.strip()

def create_user(username, password, ip, memory, cpu_cores):
    """Create a new LXC container for the user."""
    container_name = f"container_{username}"
    
    # Create the container
    run_command(f"lxc-create -t ubuntu -n {container_name}")
    
    # Start the container
    run_command(f"lxc-start -n {container_name} -d")
    
    # Wait for the container to get an IP address
    run_command("sleep 5")
    
    # Create a virtual network namespace
    run_command(f"ip netns add {container_name}")
    
    # Link the namespace to the container's network
    run_command(f"ip link add veth-{container_name} type veth peer name veth0 netns {container_name}")
    run_command(f"ip link set veth-{container_name} up")
    run_command(f"ip netns exec {container_name} ip link set veth0 up")
    
    # Assign IP to the virtual network interface
    run_command(f"ip netns exec {container_name} ip addr add {ip}/24 dev veth0")
    run_command(f"ip netns exec {container_name} ip route add default via {ip}")
    
    # Set up NAT for the namespace
    run_command(f"iptables -t nat -A POSTROUTING -s {ip}/24 -o eth0 -j MASQUERADE")
    run_command(f"iptables -A FORWARD -i eth0 -o veth-{container_name} -j ACCEPT")
    run_command(f"iptables -A FORWARD -o eth0 -i veth-{container_name} -j ACCEPT")
    
    # Set the root password
    run_command(f"lxc-attach -n {container_name} -- bash -c 'echo \"root:{password}\" | chpasswd'")
    
    # Set memory and CPU limits
    run_command(f"lxc-cgroup -n {container_name} memory.limit_in_bytes {memory}")
    run_command(f"lxc-cgroup -n {container_name} cpuset.cpus {cpu_cores}")
    
    # Install and configure SSH
    run_command(f"lxc-attach -n {container_name} -- apt-get update")
    run_command(f"lxc-attach -n {container_name} -- apt-get install -y openssh-server")
    run_command(f"lxc-attach -n {container_name} -- service ssh restart")
    
    print(f"User {username} created with IP {ip}, memory {memory}, and CPU cores {cpu_cores}.")

def delete_user(username):
    """Delete an LXC container."""
    container_name = f"container_{username}"
    ip = f"10.0.0.{100+int(container_name.split('_')[1])}"  # Assuming IPs are in the range 10.0.0.100+
    run_command(f"lxc-stop -n {container_name}")
    run_command(f"lxc-destroy -n {container_name}")
    run_command(f"ip netns delete {container_name}")
    run_command(f"iptables -t nat -D POSTROUTING -s {ip}/24 -o eth0 -j MASQUERADE")
    run_command(f"iptables -D FORWARD -i eth0 -o veth-{container_name} -j ACCEPT")
    run_command(f"iptables -D FORWARD -o eth0 -i veth-{container_name} -j ACCEPT")
    print(f"User {username} deleted.")

def update_resources(username, memory, cpu_cores):
    """Update the resources for an LXC container."""
    container_name = f"container_{username}"
    run_command(f"lxc-cgroup -n {container_name} memory.limit_in_bytes {memory}")
    run_command(f"lxc-cgroup -n {container_name} cpuset.cpus {cpu_cores}")
    print(f"Resources for {username} updated: memory {memory}, CPU cores {cpu_cores}.")

# Example usage
# create_user("user1", "password123", "10.0.0.101", "512M", "0,1")
# update_resources("user1", "1G", "0-1")
# delete_user("user1")
```

### 1.2 Save the Script to Your Repository

Save this script to your GitHub repository in a file called `manage_users.py`.

```bash
git init
git add manage_users.py
git commit -m "Add manage_users script"
git remote add origin <YOUR_GITHUB_REPO_URL>
git push -u origin main
```

## Step 2: Setting Up the Discord Bot

### 2.1 Creating the Discord Bot Script

Create another Python script `bot.py` to manage the Discord bot commands.

```python
import discord
from discord.ext import commands
from discord import app_commands
import subprocess

# Function to run shell commands
def run_command(command):
    """Run a shell command."""
    result = subprocess.run(command, shell=True, capture_output=True, text=True)
    if result.returncode != 0:
        return f"Error: {result.stderr}"
    return result.stdout.strip()

# Bot initialization
intents = discord.Intents.default()
bot = commands.Bot(command_prefix="!", intents=intents)

# Define a class to handle slash commands
class SSHManager(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
    
    @app_commands.command(name="create", description="Create a new SSH user")
    async def create_user(self, interaction: discord.Interaction, username: str, password: str, ip: str, memory: str, cpu_cores: str):
        """Create a new LXC container for the user."""
        output = run_command(f"python manage_users.py create {username} {password} {ip} {memory} {cpu_cores}")
        await interaction.response.send_message(output)

    @app_commands.command(name="update", description="Update resources for an existing SSH user")
    async def update_user(self, interaction: discord.Interaction, username: str, memory: str, cpu_cores: str):
        """Update the resources for an LXC container."""
        output = run_command(f"python manage_users.py update {username} {memory} {cpu_cores}")
        await interaction.response.send_message(output)
    
    @app_commands.command(name="delete", description="Delete an existing SSH user")
    async def delete_user(self, interaction: discord.Interaction, username: str):
        """Delete an LXC container."""
        output = run_command(f"python manage_users.py delete {username}")
        await interaction.response.send_message(output)
    
    # Adding prefix commands for the same functionalities
    @commands.command(name="create")
    async def create_user_prefix(self, ctx, username: str, password: str, ip: str, memory: str, cpu_cores: str):
        """Create a new LXC container for the user."""
        output = run_command(f"python manage_users.py create {username} {password} {ip} {memory} {cpu_cores}")
        await ctx.send(output)
    
    @commands.command(name="update")
    async def update_user_prefix(self, ctx, username: str, memory: str, cpu_cores: str):
        """Update the resources for an LXC container."""
        output = run_command(f"python manage_users.py update {username} {memory} {cpu_cores}")
        await ctx.send(output)
    
    @commands.command(name="delete")
    async def delete_user_prefix(self, ctx, username: str):
        """Delete an LXC container."""
        output = run_command(f"python manage_users.py delete {username}")
        await ctx.send(output)

# Add the cog to the bot
bot.add_cog(SSHManager(bot))

# Synchronize the slash commands with Discord
@bot.event
async def on_ready():
   

 try:
        synced = await bot.tree.sync()
        print(f"Synced {len(synced)} commands")
    except Exception as e:
        print(f"Error syncing commands: {e}")
    print(f'Logged in as {bot.user.name}')

# Run the bot
bot.run('YOUR_DISCORD_BOT_TOKEN')
```

### 2.2 Save the Bot Script to Your Repository

Save this script to your GitHub repository in a file called `bot.py`.

```bash
git add bot.py
git commit -m "Add Discord bot script"
git push
```

## Step 3: Running the Scripts

### 3.1 Cloning the Repository

Clone your GitHub repository to your VPS:

```bash
git clone <YOUR_GITHUB_REPO_URL>
cd <YOUR_REPO_NAME>
```

### 3.2 Running the Bot

Ensure that you replace `'YOUR_DISCORD_BOT_TOKEN'` with your actual Discord bot token in the `bot.py` script. Then, run the bot:

```bash
python bot.py
```

### 3.3 Using the Bot Commands

You can now use the bot to manage your LXC containers via Discord commands.

**Slash Commands:**

- `/create username password ip memory cpu_cores`
- `/update username memory cpu_cores`
- `/delete username`

**Prefix Commands:**

- `!create username password ip memory cpu_cores`
- `!update username memory cpu_cores`
- `!delete username`

## Conclusion

By following this tutorial, you have set up a system that uses LXC containers with virtual IPs managed through a Discord bot. This setup allows for easy management of users and resources on your VPS. Ensure your firewall settings and network configurations are appropriately set up to allow SSH connections to the virtual IPs. 

Feel free to extend and modify these scripts to fit your specific requirements. Happy coding!
