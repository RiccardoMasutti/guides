[ [<< Back to Main Menu](https://github.com/seth586/guides/blob/master/README.md) ]

[ [Intro](README.md) ] - [ [OpenSSH Client](1_install_client.md) ] - [ [OpenSSH on OpenWRT](2_install_openssh.md) ] - [ [Generate Keys](3_keys.md) ] - [ [Bastion](4_bastion.md) ] - **[Hardening]**

## SSH Bastion & Security Hardening Guide
### Upgrade opkg to https connections
```
root@OpenWrt:~# opkg update
root@OpenWrt:~# opkg install wget
root@OpenWrt:~# opkg install ca-certificates
root@OpenWrt:~# opkg install libustream-openssl
root@OpenWrt:~# nano /etc/opkg/distfeeds.conf
```
Replace all `http` urls with `https`. Save (CTRL+O, ENTER) and exit (CTRL+X)
```
root@OpenWrt:~# opkg update
```
You should see update lists download from https addresses and see `signature check passed`.

### Require SSH tunnel to access OpenWRT's Luci web-ui
OpenWRT's web user interface, called Luci, has nearly all privileges as a root user on the command line. Upgrading our SSH authentication to public key cryptography doesn't do us any good if the web-ui is accessible with password authentication. So lets only allow luci to be accessed from an authenticated SSH tunnel.

```
root@OpenWrt:~# nano /etc/ssh/sshd_config
```
Uncomment the following line and enable `AllowTcpForwarding`
```
AllowTcpForwarding yes
```
Save (CTRL+X, ENTER) and exit (CTRL+X). Restart `sshd` with:
```
root@OpenWrt:~# /etc/init.d/sshd restart
```

Now lets configure the client
```
User@Desktop ~ $ nano ~/.ssh/config
```
Add `LocalForward 127.0.0.1:8000 127.0.0.1:80` to your `host openwrt` block to securely tunnel client side requests on 127.0.0.1:8000 to server side 127.0.0.1:80. It should look something like this:
```
### The Bastion Host
Host openwrt
  HostName 192.168.84.1
  IdentityFile ~/.ssh/openwrt
  IdentityFile ~/.ssh/openwrt_yubi5_nano
  User root
  Port 22
  LocalForward 127.0.0.1:8000 127.0.0.1:80
### The Remote Host
Host freenas
  HostName 192.168.84.85
  IdentityFile ~/.ssh/freenas
  User root
  ProxyJump router
```
Save (CTRL+O, ENTER) and exit (CTRL+X). Now open a web browser on the client and type in `127.0.0.1:8000`. The luci web-ui should appear. Now close the SSH session, and the address should fail. Success!

Now lets restrict luci's web-ui to localhost so the only connection path is thru the SSH tunnel:
```
root@OpenWrt:~# /etc/config/uhttpd
```
Change the `http` ipv4 listen address to `127.0.0.1:80` and comment out all other listen addresses. It should look like this:
```
        # HTTP listen addresses, multiple allowed
        list listen_http        127.0.0.1:80
        #list listen_http       [::]:80

        # HTTPS listen addresses, multiple allowed
        #list listen_https      0.0.0.0:443
        #list listen_https      [::]:443
```
Save (CTRL+O, ENTER) and exit (CTRL+X). Restart the service:
```
root@OpenWrt:~# /etc/init.d/uhttpd restart
```
Now try the old address for logging in to luci (such as `192.168.84.1`) in a web browser. You should get a 'refused to connect error'. 

Now SSH into openwrt, then open a browser to `127.0.0.1:8000`. You should see the luci web-ui. Success! This process can be replicated for any web-ui that needs securing, such as RTL in the bitcoin guide!

### Require SSH tunnel to access FreeNAS WebGUI
#### Client Config
```
User@Desktop ~ $ nano ~/.ssh/config
```
Add the `localforward` rule to your `host freenas` block:
```
Host freenas
  HostName 192.168.84.85
  IdentityFile ~/.ssh/freenas
  User root
  ProxyJump router
  LocalForward 127.0.0.1:8080 127.0.0.1:80
```
Save (CTRL+O, ENTER) and Exit (CTRL+X)

#### Router Config
Make sure `TcpPortForwarding yes` is in your router's `/etc/ssh/sshd_config`.

#### FreeNAS Config
Log in to your FreeNAS WebGUI. Click "Services", "Configure" adjacent to the SSH line, click "Allow TCP Port Forwarding". Click "Save".

SSH into your freenas server:
```
root@freenas[~]# service sshd onestart
root@freenas[~]# nano /usr/local/etc/nginx/nginx.conf
```
In the `server {` block comment out and chenge the following lines:
```
        #listen                 0.0.0.0:443 default_server ssl http2;
        #listen                 [::]:443 default_server ssl http2;
        
        listen       127.0.0.1:80;
        #listen       [::]:80;
```
Save (CTRL+O, ENTER) and exit (CTRL+X). Now restart nginx:
```
root@freenas[~]# nginx restart
```
Try entering your FreeNAS IP address, it should fail. Now on your client machine, SSH into freenas, then open a browser to 127.0.0.1:8080, you should see the FreeNAS WebGUI!

Note: Until FreeNAS allows redirecting the WebGUI to `127.0.0.1` from the WebGUI's "System" > "General" settings, this configuration will likely not survive an upgrade. If you want this feature, make sure to add your voice to the issue here: https://jira.ixsystems.com/browse/NAS-104177


[ [<< Back to Main Menu](https://github.com/seth586/guides/blob/master/README.md) ]
