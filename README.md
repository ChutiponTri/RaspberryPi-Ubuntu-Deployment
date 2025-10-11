Raspberry Pi Ubuntu Deployment (Oct 2025)

This guide explains how to deploy a full-stack project on a Raspberry Pi running Ubuntu Server, including MySQL, FastAPI, Next.js, and phpMyAdmin.

---

1. Install Ubuntu Server on Raspberry Pi

- Download the Raspberry Pi Ubuntu Server image: https://ubuntu.com/download/raspberry-pi
- Follow the installation instructions and set up your user/password.

---

2. Connect Raspberry Pi to the Internet

Using LAN
- Simply connect the Raspberry Pi to your router via Ethernet.

Using Wi-Fi
1. Check your netplan configuration files:

```
$ ls /etc/netplan/
```

2. Edit the netplan YAML file (replace 50-cloud-init.yaml with your file if different):

```
$ sudo nano /etc/netplan/50-cloud-init.yaml
```

3. Example Wi-Fi configuration:

```
network:
    version: 2
    renderer: networkd
    ethernets:
        eth0:
          dhcp4: true
          optional: true
    wifis:
        wlp3s0:
            optional: true
            access-points:
                "TONX":
                    password: "late1978"
            dhcp4: true
```

> Make sure to include quotes "" around SSID and password if they contain special characters.

4. Apply the configuration:

```
$ sudo netplan apply
```
```
$ sudo netplan --debug apply
```
```
$ ip a  # Verify that the Raspberry Pi has an IP address
```

5. In case that WiFi has no password but require html login, we could do

```
network:
    version: 2
    renderer: networkd
    ethernets:
        eth0:
          dhcp4: true
          optional: true
    wifis:
        wlp3s0:
            optional: true
            access-points:
                "ENGR_Wifi": {}
            dhcp4: true
```
```
$ sudo netplan apply
```

Then command 

```
curl -v http://1.1.1.1        # Your network login ip
```

In this case got return of login route at http://192.0.2.1/login.html, so I command to login via

```
curl -d "username=USERNAME&password=PASSWORD" http://192.0.2.1/login.html
```

---

3. Setup Development Environment

- Use VSCode Remote Explorer to SSH into your Raspberry Pi for easier coding.
- Copy your project files from your computer to the Raspberry Pi.
- Ensure your docker-compose.yml is in place.  

Example phpmyadmin service for ARM64 Raspberry Pi:

```
phpmyadmin:
  image: phpmyadmin:5.2-apache
  platform: linux/arm64
  container_name: house_phpmyadmin
  restart: unless-stopped
  depends_on:
    - db
  environment:
    PMA_HOST: db
    PMA_PORT: 3306
    PMA_USER: etunmide007
    PMA_PASSWORD: Chayawut16
    MYSQL_ROOT_PASSWORD: Chayawut16
  ports:
    - "8080:80"
  networks:
    - house-demo-networks
```

---

4. Run Docker Containers

```
$ sudo docker compose up -d
```

> Note: The FastAPI app may initially fail to connect to MySQL. This is normal because the MySQL container may still be initializing.

---

5. Configure MySQL User Privileges

1. Access the MySQL container:

```
$ sudo docker exec -it house_database mysql -uroot -p
# Enter root password
```

2. Create a user and grant privileges:

```
GRANT ALL PRIVILEGES ON mydb.* TO 'userton'@'%' IDENTIFIED BY 'ton123';
FLUSH PRIVILEGES;
```

3. After flushing privileges, your FastAPI app should be able to connect.

---

6. Troubleshooting Notes

- Root user: Avoid using root for your app connections; create a dedicated user instead.
- Container restart: If MySQL was initialized without a password, restart the container and flush privileges.
- FastAPI connection: Make sure the database is fully initialized before starting FastAPI or Next.js containers.
- ARM64 compatibility: On Raspberry Pi, use phpmyadmin:5.2-apache to avoid compatibility issues.
- Network names: Ensure the FastAPI DB_HOST points to the correct Docker service name (db in the example).

---

7. Remote with Cloudflared

- Setup tunnel on Cloudflare Zero Trust as ssh://IP_ADRESS:PORT (In this case remote to Raspberry Pi at port 22)
- Setup access -> Add Self Hosted Application, make sure the application name is the same as tunnel name -> Set browser rendering option as ssh -> Add policy then change rule to email and input your email -> Then add the policy to application -> These are all require setup in cloudflared
- Then go into the link of the tunnel -> Login with username and password -> then commmand

1. Create Cloudflare Tunnel

Set up a tunnel in **Cloudflare Zero Trust Dashboard** with the following service configuration:

```
ssh://<IP_ADDRESS>:<PORT>
```

> 💡 Example: `ssh://192.168.1.10:22` (for Raspberry Pi SSH)

2. Configure Access Application

1. Go to **Access → Applications → Add an Application**  
2. Choose **Self-Hosted Application**  
3. **Application Name** — must match your tunnel name  
4. Under **Browser Rendering**, set the option to **SSH**  
5. Create an **Access Policy**:
   - Change **Rule** type to `Email`
   - Add your email address (e.g. `you@example.com`)
6. Attach the new policy to your application

3. Access the Tunnel

1. Open the tunnel link shown in your Cloudflare dashboard  
2. Log in using your configured email authentication  
3. Once authenticated, connect via SSH in your terminal:

```bash
ssh user@<your-tunnel-hostname>
```
Or, if using cloudflared locally: Verify the device with command 

```bash
cloudflared access ssh --hostname ssh.YOUR-DOMAIN-NAME.com
```

Remote Access with command
bash
```
ssh -o ProxyCommand="cloudflared access ssh --hostname %h" USER@ssh.YOUR-DOMAIN-NAME.com
```


More On Private Keys

```bash
ssh-keygen -t ed25519 -C "C-NAME" 
```
save the key file in the same directory in my case /home/pi/.ssh/KEY-NAME 

```bash
ls ~/.ssh
```

will see authorized_key, KEY-NAME, KEY-NAME.pub

```bash
more ~/.ssh/KEY-NAME.pub
```
will see ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDqC99ROz2zWRXa7pOsZbFirajxWk3lIBNaM79taRngl C-NAME

copy it and put in

```bash
sudo nano authorized_keys
```

Finally command to get private key

```bash
more ~/.ssh/KEY-NAME
```

---

8. References

- Ubuntu Wi-Fi Configuration from Command Line: https://linuxconfig.org/ubuntu-22-04-connect-to-wifi-from-command-line
- VSCode Remote SSH: https://code.visualstudio.com/docs/remote/ssh
- Docker Compose Documentation: https://docs.docker.com/compose/


