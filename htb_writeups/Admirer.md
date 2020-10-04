# Admirer

### 10.10.10.187

### Nmap Scan Results

![nmap-scan](/Users/tarak/Desktop/HackTheBox/Linux/Admirer/nmap-scan.png)

| Server IP Address | Ports Open      |
| ----------------- | --------------- |
| 10.10.10.187      | TCP: 21, 22, 80 |

### Service Enumeration

- Our nmap scan revealed a *robots.txt* file that has a disallowed entry for *admin-dir*. But we don't have access to this page. 

- After doing a lot of enumeration we see that the name specifies admin directory so we run a quick directory scan on that using dirbuster/gobuster. 

   Run the directory scan on: **http://10.10.10.187/admin-dir/**

- Also, our *robots.txt* says the directory contains personal contacts and creds in it. 

   *robots.txt* contents:

   > User-agent: *
   >
   > **This folder contains personal contacts and creds, so no one -not even robots- should see it - waldo**
   >
   > Disallow: /admin-dir

- We need to do some guess work that the directory might contain text files based on the above information provided. However, we might also try running directory scans and get lucky in finding those pages. 

- After the scans and enumeration we can find two pages we can access *credentials.txt* and *contacts.txt*. We find some useful information inside http://10.10.10.187/admin-dir/credentials.txt

   > [Internal mail account]
   > w.cooper@admirer.htb
   > fgJr6q#S\W:$P
   >
   > [FTP account]
   > ftpuser
   > %n?4Wz}R$tTF7
   >
   > [Wordpress account]
   > admin
   > w0rdpr3ss01!

- From our initial nmap-scan we know that we have an ftp service running that didn't allow anonymous login. Now, that we have our credentials we can login and continue with our enumeration. 

- #### FTP Enumeration

   - After logging into the ftp service as *ftpuser* and downloading all the information available we see mutliple backup files and **utility-scripts** directory is the one we are most interested in. 
   - 3 out of the 4 scripts work expect the *db_admin.php*. 

- **Guess work** From the machines name we need to guess that it might be using an *adminer.php* page which is used for database management. 

- Using the credentials available from our *db_admin.php* didn't work. Looking at the version used **Adminer 4.6.2** we need to find an exploit for this particular version.

### Initial shell vulnerability exploited

- #### Setting up mysql on local machine

   - By default kali doesn't allow a normal user to login as root. So, lets create a new user and login with our new credentials before setting up my sql. 

   - To create a new user and grant privileges for all the database login to mysql as root and run the following commands. 

      `CREATE USER 'hacker'@'%' IDENTIFIED BY 'password';`

      `GRANT ALL PRIVILEGES ON database_name.* TO 'hacker'@'%';`

      `flush privileges;`

   - The **%** after the user name specifies that we can connect to that user using a remote connection.  

   - To check if our new user is added run:

      `select host,user from mysql.user;`

   - Now, modify the conf file at `/etc/mysql/mariadb.conf.d/50-server.cnf` and change the *bind-address* to *0.0.0.0* to allow remote connections.

   - Restart the service and check if remote connections are allowed using:

      > service mysql restart
      >
      > mysql -u hacker -p -h IP-Address

   - Now, that everything works create a new table in the database that the user has permissions and login from the admin console. 

   - Once logged in open the database and dump the data inside the table. 

      > load data local infile '../index.php'
      >
      > into table new_table
      >
      > fields terminated by '\\n'

      - Once the query is sucessful we can now see the data dump in our table and the credentials for user *waldo*. 		

         **waldo : &<h5b~yK3F#{PaPB&dA}{H>** 

   - SSH as waldo and we now have the root flag. 

### Proof of concept code - N/A

### Privilege Escalation

- Using `sudo -l` we find that we have permissions to run `admin_tasks.sh` as root user. ![sudo-l](/Users/tarak/Desktop/HackTheBox/Linux/Admirer/sudo-l.png)

- After going through the code for this script we can see that it has multiple options and one of them executes a `backup.py` python script.

- Reading through the script we notice that we it imports a *shutil* library. Lets just hijack this python library and let it run our malicious code. 

- Create a new folder in our home directory and create a python script with the same name as `shutil.py` and create a method with the same name that our backup script imports. ![shutil](/Users/tarak/Desktop/HackTheBox/Linux/Admirer/shutil.png)

- Now, start a netcat listener on your machine and set the path for python to load the scripts.

- To set the path for python to load its libraries and run the shell script: 

   `sudo PYTHONPATH=~/path_to_root /opt/scripts/admin_tasks.sh`   

   ![Python-Path](/Users/tarak/Desktop/HackTheBox/Linux/Admirer/Python-Path.png)

- Now, we should get a reverse shell as root on our netcat listener. ![root](/Users/tarak/Desktop/HackTheBox/Linux/Admirer/root.png)

### Proof of concept code

### Key Screenshot

### Key.txt Contents
