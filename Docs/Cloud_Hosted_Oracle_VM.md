# Cloud hosted Oracle Virtual Machine

Oracle provides "Always free" ressources, including the creation of a free Virtual Machine:

* **[Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)** : _"All tenancies get the first 3,000 OCPU hours and 18,000 GB hours per month for free for VM instances using the VM.Standard.A1.Flex shape, which has an Arm processor. **For Always Free tenancies, this is equivalent to 4 OCPUs and 24 GB of memory.**"_

**Here's a great tutorial on how to get yours: [How To Set Up and Run a (Really Powerful) Free Minecraft Server in the Cloud](https://blogs.oracle.com/developers/post/how-to-set-up-and-run-a-really-powerful-free-minecraft-server-in-the-cloud#create-a-virtual-machine-instance).**

> This article explains how to set up an Oracle VM for hosting a Minecraft server, but you can use it for anything you want.

## Credit card information

Please note that you will have to provide your credit card information to access your "Always free" resources.

Even after the 30-day trial period, **you will not be charged unless you choose to upgrade your plan**, and you will still have access to your "Always Free" account.

## "Out of capacity" error message

You will probably get an error message when creating your free VM on the Oracle dashboard, saying something like this:
* _"Out of capacity for shape VM.Standard.A1.Flex in availability domain AD-1. Create the instance in a different availability domain or try again later. If you specified a fault domain, try creating the instance without specifying a fault domain, otherwise try creating the instance in a different availability domain. If that doesn’t work, please try again later."_

A workaround is to automate the click on the "Create" button. To do this, you can run this script in your browser's console:

```js
// Open the home page in a new tab
win1 = window.open("https://cloud.oracle.com/");
var v = document.querySelector(".oui-savant__Panel--Footer > button:nth-child(1)");

// At every 30 seconds, reload the home page and click "Create"
timer1 = setInterval(() => {
    win1.location.reload();
    console.log("Refreshed");
    
    if (v && v.textContent == "Create") {
        v.click();
        console.log("!!! clicked");
    } 
    else {
        console.log("!!! no button");
    }
}, 30000)
```

> Source : https://www.reddit.com/r/oraclecloud/comments/zf0tje/comment/la6qkaa/

* It automatically clicks every 30 seconds.
* It opens a new window to keep your session alive, so you don't log out.

**Please note that this can take a while, it took me about 2 hours to get it working.**

## Reclaim 15GB of `/var/oled`

Source: https://www.reddit.com/r/oraclecloud/comments/ywwp41/reclaiming_10gb_varoled/

You can see that 15GB are completely unusable with `lsblk`:
```sh
[opc@your-machine ~]$ lsblk

NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                  8:0    0 46.6G  0 disk
├─sda1               8:1    0  100M  0 part /boot/efi
├─sda2               8:2    0    2G  0 part /boot
└─sda3               8:3    0 44.5G  0 part
  ├─ocivolume-root 252:0    0 29.5G  0 lvm  /
  └─ocivolume-oled 252:1    0   15G  0 lvm  /var/oled
```

To reclaim them, run:
* `sudo umount /var/oled`
    * If "target is busy", run `sudo dnf remove pcp` before.
* `sudo dnf remove pcp`
* `sudo umount /var/oled`
* `sudo mkdir /var/oled/crash`
* `sudo lvremove /dev/mapper/ocivolume-oled`
* `sudo xfs_growfs -d /dev/mapper/ocivolume-root`

Now run `df -h /`, you should have successfully reclaimed your 15GB:
```sh
[opc@your-machine ~]$ df -h /
Filesystem                  Size  Used Avail Use% Mounted on
/dev/mapper/ocivolume-root   45G   12G   34G  26% /
```

Then edit `/etc/fstab`:
* `sudo nano /etc/fstab`
* Remove the line containing "/var/oled", or add a `#` at the beggining to comment it out.


## Firewall (with firewalld)

Open a port publicly:
* Open a port (TCP and UDP):
    ```sh
    sudo firewall-cmd --permanent --zone=public --add-port=25565/tcp

    sudo firewall-cmd --permanent --zone=public --add-port=25565/udp
    ```
* Cancel:
    ```sh
    sudo firewall-cmd --permanent --zone=public --remove-port=25565/tcp

    sudo firewall-cmd --permanent --zone=public --remove-port=25565/udp
    ```
* See all open ports:
    ```sh
    sudo firewall-cmd --zone=public --list-ports
    ```

Reject an IP adress:
* Apply :
    ```sh
    sudo firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='51.15.34.47' reject"
    ```
* Cancel :
    ```sh
    sudo firewall-cmd --permanent --remove-rich-rule="rule family='ipv4' source address='51.15.34.47' reject"
    ```
    > _This is a real abusing IP example: https://www.abuseipdb.com/check/51.15.34.47_

Reload **(REQUIRED)**:
```sh
sudo firewall-cmd --reload
```

Firewall status:
* Check if firewalld is active:
    ```sh
    sudo systemctl status firewalld
    ```
* Get firewalld config:
    ```sh
    sudo firewall-cmd --list-all
    ```

## Additional information

### A. Install/Update JDK

The [Todd Sharp documentation](https://blogs.oracle.com/developers/post/how-to-set-up-and-run-a-really-powerful-free-minecraft-server-in-the-cloud#create-a-virtual-machine-instance) also explains how to install Java with `yum` commands, but you may already have a version installed on your VM.

Here's an updated version of his tutorial:

* Check online available versions of JDK:
    ```
    yum list jdk*
    ```

* Install the latest JDK (headless):
    ```sh
    sudo yum install jdk-22-headless.aarch64
    ```

* List all installed JDK:
    ```sh
    yum list installed | grep jdk*
    ```

* Remove unused JDK:
    ```sh
    sudo yum remove jdk-17-headless.aarch64
    ```

* Check your current Java version:
    ```sh
    java --version
    ```

### B. Create a Minecraft server network

If you're interested in setting up a Minecraft server, I strongly recommend checking out **[Crafty Controller](https://craftycontrol.com/)**, **[Velocity Proxy](https://papermc.io/software/velocity)** and **[PaperMC Server](https://papermc.io/software/paper)**.

* _Paper is a Minecraft server forked from Spigot._
* _Velocity is a Minecraft server proxy and a way better alternative to BungeeCord._
* _Crafty Controller is an administration panel to create and manage Minecraft servers easily._

For more information, I have done some research and written documentation about Velocity and Crafty Controller in this repository: **[AmbersoneDream/ProxyServerResearch](https://github.com/AmberstoneDream/ProxyServerResearch/tree/main)**.
