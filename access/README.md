# Remote Access


# Navigation
- [Apps](/apps/)
- [Media Server](/media/)
- [Remote Access](/access/)
  - [Tailscale](#tailscale)
    - [How to Setup Tailscale](#how-to-setup-tailscale)
      - [Setup Tailscale Part One](#setup-tailscale-part-one)
      - [Setup Tailscale Part Two](#setup-tailscale-part-two)
      - [Tailscale Clients](#tailscale-clients)

# Tailscale

I discovered tailscale not long after the start of my homelab. It has been an amazing tool and service for all my homelab needs.

### Why Tailscale?

Tailscale made accessing my home network and my homelab 100x easier and faster. Since it's based on wireguard this makes it more like a VPN. But its not just a VPN, it is more like a mesh network. I can have 10 different networks with 10 different tailscale nodes and i can access any service on any one of them, securely.

### How it works in my setup

I use tailscale 24/7 for remote access to my home network, however I do NOT use it as an exit node (unless necessary). Having my subnet advertised by my Proxmox LXC makes this so much easier. Since i have NGINX proxy manager for local SSL/Internal Domains to my services, i can use those anywhere as if it was public, but its safely guarded inside my network. No ports, No tunnels. Its magic.

# Setup Tailscale Part One

## Step 1: 

Create Your Tailnet.

Head over to [Tailscale](https://tailscale.com) and create your account. 

**NOTE: Whatever email you use to sign up with will be your tailnet name unless you change it. It is complicated to change it after creation.**


## Step 2: 

You should be at your Tailscale admin dashboard, you will see no machines on your Tailnet.

To add your first machine, Click **Add Device**. You will see two options: **Linux Server** and **Client Device**. Lets start with Linux Server.


**1. Set up device**

I don't use tags for ACLs currently but you can configure them here if you would like. It is not required.

**Ephemeral:** Off (When enabled, Tailscale will automatically remove the device from your tailnet when it goes offline. If you restart your system it would be removed and you would need to re-add it all over again.)

**Use as exit node:** On (Optional, This means that you can use this device like a VPN service and route all your client traffic through your network. If installed on a homelab this means you would always route traffic through your home network when away.)


**2. Set up authentication key**

**Reusable:** Off (If enabled, you could use the same key to join the tailnet.)

**Auth key expiration:** Default is 7 days. (unless you have a specific reason to have temporary access, keep this at default. More on this after.)

**3. Generate install script**

Click generate install script. This will give you a simple copy and paste command to enter anywhere in your linux machine homelab. (If you use windows server or another OS that isn't linux you can run the Tailscale docker container OR use the app for windows.)

Paste this script and wait for it to connect.


## Step 3:

You should now see your new device on your Tailnet. If successful move onto [Part Two](#part-two)

**NOTE: Make sure you click the ••• beside your machine and select "Disable Key Expiry" or else in 7 days your machine will need to be reauthenticated.**


# Setup Tailscale Part Two


# Configuring Subnet Routing with Tailscale

### Step 1: Identify the Subnet

Identify the subnet you want to route through Tailscale, e.g., `192.168.2.0/24`.

### Step 2: Enable IP Forwarding

Open the `sysctl.conf` file to enable IP forwarding:

```
sudo nano /etc/sysctl.conf
```

Uncomment or add the following line:

```
net.ipv4.ip_forward = 1
```

Apply the changes immediately:

```
sudo sysctl -p
```

### Step 3: Configure Tailscale for Subnet Routing

Run the following command to advertise routes:

```
sudo tailscale up --advertise-exit-node --advertise-routes=192.168.2.0/24
```

> **Note:** Replace `192.168.1.0/24` with your specific subnet.

### Step 4: Authorize the Route in Tailscale Dashboard

1. **Head back to the Tailscale Dashboard**:  
   Navigate to [login.tailscale.com](https://login.tailscale.com).

2. **Find your Device**:  
   Go to the "Machines" section and locate your machine.

3. **Review Offered Routes**:  
   Click on the device to view offered routes under "View route settings".

4. **Enable the Route**:  
   Check the box next to the route to authorize and enable it.

### Step 5: Verify the Configuration

Ensure that devices connected to Tailscale can access the advertised subnet by pinging an IP within it:

```
ping 192.168.2.x
```

> **Note:** Replace `192.168.2.x` with a valid IP address in your subnet.


# Tailscale Clients

Now that you have Tailscale on your server configured. It's time to set up the mobile client or PC client.

### For iOS

[Download the Tailscale App](https://apps.apple.com/ca/app/tailscale/id1470499037)

Then sign in to your account used to create your Tailnet earlier. To access your homelab just turn on the VPN toggle, allow Tailscale to make VPN connections when prompted. You now have access to your home network when away from home.


### For Windows

Head over to the [Tailscale Downloads](https://tailscale.com/download) page and choose Windows. Follow the setup process and sign in to complete setup. 


### Other OS

Visit [Tailscale Downloads](https://tailscale.com/download) to get the proper installation for your device of choice.
