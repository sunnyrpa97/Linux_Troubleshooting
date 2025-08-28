# Troubleshooting NXDOMAIN Errors on Linux 🛠️

Got an `NXDOMAIN` error on your Linux laptop after running `nslookup`? It means your system can't find the IP address for a domain you're trying to reach. Don't worry, it's usually a local DNS configuration problem. Here’s a quick guide to fix it.

-----

## 1\. Check `/etc/resolv.conf`

This is almost always the first place to look. This file tells your system which DNS servers to use.

  * **See what's inside:**

    ```bash
    cat /etc/resolv.conf
    ```

  * **What you're looking for:** Check the IP addresses next to `nameserver`. They might be wrong, missing, or pointing to a broken local DNS cache (like `127.0.0.53`).

  * **Quick Test (Temporary Fix):** You can manually add a public DNS server to see if it fixes the problem.

    ```bash
    sudo nano /etc/resolv.conf
    ```

    Add this line, then save the file:

    ```
    nameserver 8.8.8.8
    ```

    Try your `nslookup` command again. **Heads up:** This change will likely be erased on reboot or network reconnection.

-----

## 2\. Test a Specific DNS Server ✅

This step bypasses your system's settings to see if the problem is your configured DNS server or a bigger network issue.

  * **Run `nslookup` with a public DNS server:**
    ```bash
    nslookup google.com 8.8.8.8
    ```
  * **How to read the result:**
      * **If it works:** The problem is definitely with the servers listed in your `/etc/resolv.conf`. Jump down to [Step 5](https://www.google.com/search?q=%235-make-your-dns-changes-permanent) to fix it for good.
      * **If it fails:** The issue is probably with your network connection or a **firewall**. Keep going to the next step.

-----

## 3\. Check Network & Firewall Settings 🌐

Let's make sure you're actually connected to the internet and nothing is blocking DNS traffic.

  * **Check your IP address and route:**

    ```bash
    ip addr show
    ip route show
    ```

    Make sure you have an IP address and a default route.

  * **Ping a public IP:** This checks for internet access without using DNS.

    ```bash
    ping -c 4 8.8.8.8
    ```

  * **Check your firewall:** DNS requests use **port 53** (mostly UDP). Ensure your firewall (`ufw`, `firewalld`, etc.) isn't blocking it.

    ```bash
    # For ufw (Ubuntu-based systems)
    sudo ufw status

    # For firewalld (Fedora/CentOS-based systems)
    sudo firewall-cmd --list-all
    ```

-----

## 4\. Examine `nsswitch.conf`

This file controls the order your system uses to look up hostnames.

  * **Check the `hosts` line:**
    ```bash
    grep "hosts:" /etc/nsswitch.conf
    ```
  * **What you want to see:** The line should look something like `hosts: files dns`. This tells the system to check local files first, then use **DNS**. If `dns` is missing, that's your problem.

-----

## 5\. Make Your DNS Changes Permanent

If you found that your DNS servers were the issue, you need to save your changes permanently so they don't get overwritten.

  * **For `systemd-resolved` (Modern systems like Ubuntu):**
    1.  Edit the config file:
        ```bash
        sudo nano /etc/systemd/resolved.conf
        ```
    2.  In the `[Resolve]` section, add your DNS servers:
        ```ini
        [Resolve]
        DNS=8.8.8.8 1.1.1.1
        ```
    3.  Restart the service to apply the change:
        ```bash
        sudo systemctl restart systemd-resolved
        ```
  * **For NetworkManager (Most desktop environments):**
    It's easiest to use the GUI.
    1.  Open your **Network Settings**.
    2.  Select your active connection (Wi-Fi or Ethernet) and open its settings.
    3.  Go to the **IPv4** tab.
    4.  Change the method to **"Automatic (DHCP) addresses only"**.
    5.  In the **DNS servers** field, type `8.8.8.8, 1.1.1.1`.
    6.  Save, then turn your connection off and on again.