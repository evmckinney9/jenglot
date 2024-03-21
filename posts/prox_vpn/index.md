---
title: Configuring an LXC container with an external VPN
description: |
    This guide explains how to integrate your LXC container with an external VPN service to enhance the privacy and security of your containerized applications.
categories: [proxmox, devops]
date: 3/20/2024
code-line-numbers: false
---

# Objective

This guide explains how to integrate your LXC container with an external VPN service to enhance the privacy and security of your containerized applications.

- **Proxmox Virtual Environment**: A server management platform that allows you to deploy, manage, and monitor LXC containers. For more information visit [Proxmox VE](https://www.proxmox.com/proxmox-ve).

- **TTeck's Proxmox Helper Scripts**: These scripts make it easier to create and manage LXC containers. Learn more [here](https://tteck.github.io/Proxmox/).

In this guide, we will configure a qBittorrent LXC container to use WireGuard VPN as its network gateway. I selected AirVPN as the VPN provider due to its support for peer-to-peer (P2P) connections and port forwarding capabilities. However, the concepts and steps outlined here can be adapted to other VPN services and applications according to your needs.

::: {.callout-warning}
Please use this guide responsibly. I'm not an expert and cannot guarantee the security of your setup.
:::

# Setup

## WireGuard LXC

First, set up WireGuard LXC to integrate your container with the VPN:

1. **Installing WireGuard LXC**: Start by executing the script on your host machine and accept the default settings.

    ```bash
    bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/ct/wireguard.sh)"
    ```

    ![Figure 1: Initiating WireGuard LXC setup.](figures/image-18.png)  

    After running the script, install necessary dependencies:

    ```bash
    apt install openresolv
    apt-get install iptables-persistent
    ```

2. **Acquiring the VPN Configuration**:
   - Go to [AirVPN's WireGuard configuration generator](https://airvpn.org/linux/wireguard/terminal/).

    ![Figure 2: Generating AirVPN WireGuard configuration.](figures/image-19.png)  

   - Select Linux, the WireGuard protocol, and your preferred server location. This will generate a *.conf file with your VPN configuration.

    ```conf
    [Interface]
    Address = 10.178.17.78,fd7d:76ee:e68f:a993:124c:7d1d:869c:84e7
    PrivateKey = [PRIVATE_KEY]
    MTU = 1320
    DNS = 10.128.0.1, fd7d:76ee:e68f:a993::1

    [Peer]
    PublicKey = PyLCXAQT8KkM4T+dUsOQfn+Ub3pGxfGlxkIApuig+hk=
    PresharedKey = [PRIVATE_KEY]
    Endpoint = america3.vpn.airdns.org:1637
    AllowedIPs = 0.0.0.0/0,::/0
    PersistentKeepalive = 15
    ```

3. **Configuring WireGuard with AirVPN**:
   - Check your current public IP address:

    ```bash
    curl ifconfig.me
    ```

   - Update the WireGuard configuration with AirVPN's details (e.g., `wg0.conf`, but you can use any name if you are setting up multiple VPNs on the same LXC):
    ```bash
    nano /etc/wireguard/wg0.conf
    ```

   - Apply the new configuration and restart WireGuard:

    ```bash
    wg-quick down wg0
    wg-quick up wg0
    wg show
    ```

   - Confirm the VPN is working by checking your new public IP address:
    ```bash
    curl ifconfig.me
    ```

    You should observe an IP address different from the one initially confirmed, indicating successful VPN integration.

    ![Figure 3: Verifying WireGuard VPN functionality](figures/image-20.png)  

## qBittorrent LXC

Next, set up qBittorrent LXC for secure torrenting through the VPN:

1. **Installing qBittorrent LXC**: Start by executing the script on your host machine and accept the default settings.

    ```bash
    bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/ct/qbittorrent.sh)"
    ```

    ![Figure 4: Launching the qBittorrent LXC setup.](figures/image-2.png)  

    After installation, access qBittorrent through your web browser at the provided URL:

    ```shell
    qBittorrent should be reachable by going to the following URL.
             http://192.168.0.227:8090
    ```

## Network bridge

Creating a network bridge in Proxmox ensures direct communication between the WireGuard and qBittorrent containers:

1. **Creating a Proxmox Network Bridge**:
   - Start by creating a network bridge in the Proxmox interface, which may vary based on your setup.

    ![Figure 6: Configuring a new network bridge in Proxmox.](figures/image-6.png)  

    - Apply the changes to activate the new network bridge.

    ![Figure 7: Finalizing network bridge setup.](figures/image-5.png)  

2. **Integrating LXCs with the Network Bridge**:
   - Assign the new bridge as a network device for both the WireGuard and qBittorrent LXCs, setting static IP addresses on this new subnet:

     - WireGuard LXC: `10.10.10.1/24`
     - qBittorrent LXC: `10.10.10.2/24`

    ![Figure 8: Assigning the network bridge to LXCs.](figures/image-7.png)  

   - Test the integration by pinging the qBittorrent LXC from the WireGuard LXC.

    ```bash
    ping 10.10.10.2
    ```

    ![Figure 9: Verifying connectivity between LXCs.](figures/image-8.png)  

3. **Gateway Configuration**:

    - **WireGuard LXC**:
        - Enable IP forwarding and establish NAT rules:
        ```bash
        echo "net.ipv4.ip_forward=1" | tee -a /etc/sysctl.conf
        sysctl -p
        iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
        ```
        - Save the iptables configuration to persist across reboots:
            ```bash
            netfilter-persistent save
            ```

    - **qBittorrent LXC**:
        - Define the WireGuard LXC as the default gateway. Update `/etc/network/interfaces` with the gateway and DNS settings.

        ```bash
        auto lo
        iface lo inet loopback

        auto eth0
        iface eth0 inet static
            address 192.168.0.227 # [Your qBittorrent LXC's static IP]
            netmask 255.255.255.0

        auto eth1
        iface eth1 inet static
            address 10.10.10.2/24
            netmask 255.255.255.0
            gateway 10.10.10.1
            dns-nameservers 1.1.1.1
            post-up ip route add default via 10.10.10.1 dev eth1
            post-up ip route del default via 192.168.0.1 dev eth0 || true
        ```
        
      ::: {.callout-tip}
I recommend using a static IP address for the qBittorrent LXC as DHCP can cause issues with the setup's persistence across reboots.
:::

        ![Figure 10: Default routing through the VPN bridge.](figures/image-10.png)  

        - Restart network services to apply changes.

        - Confirm the successful gateway configuration by verifying the IP address is now resolved through the VPN:

        ```bash
        ping -c 4 google.com  # Test DNS resolution
        curl ifconfig.me  # Should return the WireGuard IP
        ```

        - In case of any configuration issues, try bypassing Proxmox's configuration checks:

        ```bash
        touch /.pve-ignore.resolv.conf
        touch /etc/network/.pve-ignore.interfaces
        ```

## qBittorrent client configuration

Finally, configure the qBittorrent client to ensure secure torrenting through the VPN:

1. **Port Forwarding with AirVPN**: Visit [AirVPN's port forwarding section](https://airvpn.org/ports/) to obtain a port to be forwarded through the VPN. This step is crucial for the qBittorrent client to establish direct connections with peers.

    ![Figure 6: Acquiring a forwarded port from AirVPN.](figures/image-13.png)  

2. **qBittorrent WebUI**:
   - Ensure the security of your qBittorrent interface by setting a strong password.

    ![Figure 5: Configuring the webgui password for qBittorrent.*](figures/image-4.png)  

3. **Configuring the Listening Port**: Access the settings in the qBittorrent WebUI and enter the listening port you received from AirVPN. 

    ![Figure 7: Setting the listening port in qBittorrent.](figures/image-14.png)  

4. **Network Interface Binding**:
   - Bind qBittorrent to the network interface (`eth1`) and IP address (`10.10.10.2`) corresponding to the VPN connection to ensure all traffic goes through the VPN.

    ![Figure 8: Binding qBittorrent to a specific network interface.](figures/image-12.png)  

- Remember to click **Save** to apply your changes.

# Verification

Follow these steps to confirm everything is functioning as intended:

1. **Torrent Address Detection**: Use [ipleak.net](https://ipleak.net/) to check that the IP address of your torrent client matches that of the VPN.

    ![Figure 9: Verifying the VPN IP address with ipleak.net.](figures/image-15.png)  

2. **Torrent Download Test**: For a reliable test, I use a latest [Arch Linux ISO torrent](http://mirror.cs.pitt.edu/archlinux/iso/latest/archlinux-2024.03.01-x86_64.iso.torrent) supplied by my university. Successful downloading confirms that your qBittorrent client is properly communicating with peers through the VPN.
    ![Figure 10: Confirming successful torrent download.](figures/image-17.png)  

With these steps completed, your LXC container should now be securely integrated with a VPN.
