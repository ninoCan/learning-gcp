# Google Cloud Environment

We are now going to familiarize with the use of GCE services.

## Launching two virtual machines instances

This is done through the GUI on the page [computer
engine](instances?project=dev2ops&instancessize=50). There is a "+" button at the top, click it.
Then we add the instance name, for the scope of our project Dev2Ops course, we will call them
"n5602-backend" and "n5602-database" where _n5602_ is my personal student number (you should use
yours).
Then, select from the drop-down  **Region** menu the option "europe-west1 (Belgium)", from the menu
**Machine type** select "f1-micro". Then scroll down and click "Change" in the **Boot disk**
section. Scroll down and tick both "Allow HTTP traffic" and "Allow HTTPS traffic" boxes in the
**Firewall** section. Now you can scroll till the end and "Create".

After these steps, we successfully fired up two machinces that will be listed amongst the other
student's VMs; notice the Internal IP and external IP have been automatically created by Google.
For my personal VMs I have the following:

| VM name  | Internal IP | External IP   | 
| ----:    | :---:       | :---:         | 
| backend  | 10.132.0.42 | 34.77.105.216 | 
| database | 10.132.0.43 | 35.233.19.119 | 

ACHTUNG: Following our settings, the external IP is "ephemeral": this means  they will change when
we restart the server; this will be covered later in static IP section. 
We can access them by clicking the SSH button on the left.

## Inside the VMs

Let us start by installing Docker on the two machinces:

    sudo apt-get update
    sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update && sudo apt-get install docker-ce docker-ce-cli containerd.io
Let us give docker the sudo permission with:
    
    $ sudo usermod -aG docker "$USER"
    $ newgrp docker
Next, on the backend vm let us pull the docker image, we already uploaded to the GCE registy:

    $ gcloud docker -- pull eu.gcr.io/<project-id>/<image-name>
In my case the path of the image is going to be `eu.gce.io/dev2ops/n5602-lapsrv:1.0`.
Let us now start it with the `docker run` command:

    $ docker run -e DB_NAME=wordpress -e DB_USER=dbuser -e DB_PASSWORD=dbpass -e DB_HOST=10.132.0.43
        -p 80:80 --name=backend -d eu.gcr.io/dev2ops/n5602-lapsrv:1.0
Let now create the database, in the week about docker we just used the mysql image, let us do
the same now. To begin with, create a volume for the container to store the database information.

    $ docker volume create crv_mysql
Then, launch a container with the official mysql image:

    $ docker run -d --rm --name=database -e MYSQL_USER=dbuser -e MYSQL_DATABASE=wordpress -e
    MYSQL_PASSWORD=dbpass -e MYSQL_ROOT_PASSWORD=root --mount
    type=volume,src=crv_mysql,dst=/var/lib/mysql -p 3306:3306 --rm -d mysql:5.7 mysqld --innodb-buffer-pool-side=64M
the last part of the command (`mysqld --innodb-buffer-pool-side=64M`) is needed because otherwise
the image runs out of RAM time of container launch.

## Allow TPC traffic from database to backend

We will be following the [google cloude
docs](https://cloud.google.com/vpc/docs/using-firewalls#serviceaccounts) to open up the port 3306
used by mysql to communicate the database data.

To 
## Reserve a static IP
(INCOMPLETE: google the title! and maybe change it...)
We set the backend ip to a static value:

| name  | External Address |
| --- | --- | 
| n5602 |  104.199.48.249  |


## Securing the webpage with SSL certificate

### Setting things up
We will obtain the SSL certificates through [Let's
Encrypt](https://letsencrypt.org/getting-started/).
Let us install the letsencrypt(Certbot) client:
    
    sudo add-apt-repository -y ppa:certbot/certbot
    sudo apt-get update
    sudo apt-get install -y certbot
<!-- $ sudo apt-get update && sudo apt-get install letsencrypt -->
Since we do not want to modify the image of the container, we instead install a reverse proxy
provider: HAProxy. This will redirect the traffic incoming to the External IP of our server to the
container. To do so we will have to modify the container we launched to redirect to HAProxy.
Let's install HAProxy:
    
    $ sudo apt-get install haproxy
Now, let us relauch the container with the guest port 80 connected to the host subport 3000 on the
loopback address: 127.0.0.1:3000 with:

    $ docker run -e DB_NAME=wordpress -e DB_USER=dbuser -e DB_PASSWORD=dbpass -e DB_HOST=10.132.0.43
    -p 127.0.0.1:3000:80 --name=backend -d eu.gcr.io/dev2ops/n5602-lapsrv:1.0

### Obtain certificates

Now, we are going to obtain the a SSL certificate through a letsencrypt plugin, also known as
"authenticator" because it is used to authenticate whether a server should be issued a certificate.
In order for it to work, let us stop first the HAProxy service with:

    $ sudo service haproxy stop
Verify that port 80 is not in use with:

    $ netstat -na | grep ':80.*LISTEN'
which should return no output. To obtain the certificate let us run the Standalone plugin (certbot):

    $ sudo certbot certonly --standalone --preferred-challenges http --http-01-port 80 -d n5602.my-project.dev 
Where I used my student number, make sure that you substitute yours. You will be asked to provide
your email for renewal and security notices; and after it to agree with the _Terms of service_.

The certificates and your private keys  are placed in the directory `/etc/letsencrypt/archive/` and
symbolic links should have been automatically created to folder `/etc/letsencrypt/live/`; this
folders belong to root, so you need sudo to access them. HAProxy will need  to use the
`fullchain.pem` and the `privkey.pem` to transmit encrypted traffic; let us combine these into a
single file. Create a folder to store this file with:

    $ sudo mkdir -p /etc/haproxy/certs
and let's combine the two files with:

    $ DOMAIN='n5602.my-project.dev' sudo -E bash -c 'cat /etc/letsencrypt/live/$DOMAIN/fullchain.pem /etc/letsencrypt/live/$DOMAIN/privkey.pem > /etc/haproxy/certs/$DOMAIN.pem'
ONCE AGAIN: make sure to initialize the variable DOMAIN with your own personal domain. Then, remove
permission to access this file to any other user but root with:

    $ sudo chmod -R go-rwx /etc/haproxy/certs
Let's create a configuration file for HAProxy as follows:

    $ sudo nano /etc/haproxy.cfg
With the following content:

    global
    daemon
    maxconn 4096

    defaults
        mode http
        timeout connect 5000ms
        timeout client 50000ms
        timeout server 50000ms

    frontend http-in
        bind *:80
        acl is_site1 hdr_end(host) -i n5602.my-project.dev
        
        use_backend site1 if is_site1
    

    backend site1
        balance roundrobin
        option httpclose
        option forwardfor
        server s2 127.0.0.1:3000 maxconn 32


    listen admin
        bind 127.0.0.1:8080
        stats enable
<!-- https://cloud.google.com/load-balancing/docs/ssl-certificates -->
http://oskarhane.com/haproxy-as-a-static-reverse-proxy-for-docker-containers/
https://certbot.eff.org/instructions
https://serversforhackers.com/c/letsencrypt-with-haproxy
https://www.digitalocean.com/community/tutorials/how-to-implement-ssl-termination-with-haproxy-on-ubuntu-14-04
