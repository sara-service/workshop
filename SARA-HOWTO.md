# Install & Configure SARA Server using bwCloud

## Intro

This manual provides a step-by-step setup for a fully configured instance of SARA. 
It is advised to walk through this manual without interruptions or intermediate reboots.

About SARA:
https://sara-service.org

In case of questions please contact:
* Stefan Kombrink, Ulm University, Germany / e-mail: stefan.kombrink[at]uni-ulm.de
* Matthias Fratz, University of Constance / email: matthias.fratz[at]uni-konstanz.de
* Franziska Rapp, Ulm University, Germany / e-mail: franziska.rapp[at]uni-ulm.de


## Requirements

You will need
* a DSpace-Repository
* an Archive GitLab

You might want a GitLab as source.
Alternatively you can use GitHub.

## Setup 

### Connect to the machine using your credentials (ssh, putty)
Please use the credentials handed out to you, e.g.

Linux/MacOS:
```bash
ssh ubuntu@vm-XXX-XXX.bwcloud.uni-ulm.de
```
Windows: Download [putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) and install it

## Get Resources
```bash
cd $HOME

# SARA source code
git clone https://git.uni-konstanz.de/sara/sara-for-workshop-with-submodules SARA-server
# Workshop materials
git clone https://github.com/sara-service/workshop.git
```

## Installation
### Postgres
```bash
sudo apt-get install postgresql
sudo systemctl start postgresql
sudo -u postgres createuser -l -D -R -S sara
sudo -u postgres psql -c "ALTER USER sara WITH PASSWORD 'secret';"
sudo -u postgres createdb -E UTF8 -O sara sara

sudo -u postgres psql -d sara -f ~/SARA-server/saradb/adminconfig.sql
sudo -u postgres psql -d sara -f ~/SARA-server/saradb/schema.sql
sed "s/__USERNAME__/sara/g" ~/SARA-server/saradb/permissions.sql | sudo -u postgres psql -d sara
sudo -u postgres psql -d sara -f ~/SARA-server/saradb/licenses.sql
```
```
DBBASEDIR="$HOME/SARA-server/saradb"
for file in $DBBASEDIR/workshop/*.sql; do
    sed -f $DBBASEDIR/credentials/workshop.sed "$file" | sudo -u postgres psql -v ON_ERROR_STOP=on -d sara -v "basedir=$DBBASEDIR";
done
```

### Apache
```bash
sudo apt install apache2 letsencrypt
sudo systemctl stop apache2
HN=$(hostname -f)
```
Install Redirect
```bash
cat << EOF | sudo tee /etc/apache2/sites-available/redirect.conf
<VirtualHost *:80>
    ServerName $HN
    ServerAdmin webmaster@localhost

    Alias "/.well-known" "/var/www/letsencrypt/.well-known"
    <Directory /var/www/letsencrypt/.well-known>
        Options -MultiViews
        Require all granted
    </Directory>

    RedirectPermanent / "https://$HN/"
</VirtualHost>
EOF
```
Install TomCat Proxy
```bash
cat << EOF | sudo tee /etc/apache2/sites-available/proxy.conf
<VirtualHost *:443>
    ServerName $HN
    ServerAdmin webmaster@localhost

    <Location />
        ProxyPass "ajp://localhost:8009/SaraServer/"
        ProxyPassReverseCookiePath "/SaraServer" "/"
    </Location>
    Alias "/.well-known" "/var/www/letsencrypt/.well-known"
    <Location /.well-known>
        ProxyPass !
    </Location>
    <Directory /var/www/letsencrypt/.well-known>
        Options -MultiViews
        Require all granted
    </Directory>

    # limit scripts, styles and fonts to same server only.
    # for images, allow local images, and https:* and data:* for logos.
    # disallow everything else.
    Header always set Content-Security-Policy "default-src 'none'; \
        script-src 'self'; style-src 'self' 'unsafe-inline'; \
        img-src 'self' https: data:; connect-src 'self'; font-src 'self'"
    # disallow frames (anti-clickjacking)
    Header always set X-Frame-Options deny
    # make sure XSS protection doesn't mess up ("sanitize") URLs
    Header always set X-Xss-Protection "1; mode=block"
    # turn of content type autodetection misfeature (major security risk)
    Header always set X-Content-Type-Options nosniff
    # turn of referrer for privacy
    Header always set Referrer-Policy no-referrer

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
	
    Alias "/.well-known" "/var/www/letsencrypt/.well-known"
    <Location /.well-known>
        ProxyPass !
    </Location>
    <Directory /var/www/letsencrypt/.well-known>
        Options -MultiViews
		Require all granted
	</Directory>

	SSLEngine on
	SSLCertificateFile /etc/letsencrypt/live/$HN/fullchain.pem
	SSLCertificateKeyFile /etc/letsencrypt/live/$HN/privkey.pem
</VirtualHost>
EOF
```
```bash
sudo mkdir -p /var/www/letsencrypt
sudo letsencrypt certonly --standalone -w /var/www/letsencrypt -d $HN
sudo a2dissite 000-default
sudo a2enmod proxy_ajp ssl headers
sudo a2ensite redirect proxy
sudo systemctl restart apache2
```

### Tomcat
```bash
sudo apt-mark hold openjdk-11-jre-headless
sudo apt-get install openjdk-8-jdk tomcat8 maven haveged

cat << EOF | sudo tee /etc/tomcat8/server.xml
<?xml version='1.0' encoding='utf-8'?>
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <!-- Prevent memory leaks due to use of particular java/javax APIs-->
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <Service name="Catalina">
    <!-- AJP Connector on port 8009, configured analogous to the default HTTP connector. -->
    <Connector port="8009" address="127.0.0.1" protocol="AJP/1.3"
		connectionTimeout="20000" URIEncoding="UTF-8"
		redirectPort="8443" />
    <Engine name="Catalina" defaultHost="localhost">
      <!-- unpackWARs=true is needed on tomcat8; unpackWARs=false is ~50x slower -->
      <Host name="localhost" appBase="webapps" unpackWARs="true" autoDeploy="true">
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access" suffix=".log" fileDateFormat=""
               pattern="%h %l %u %t &quot;%r&quot; %s %b"
               rotatable="false" checkExists="true" />
      </Host>
    </Engine>
  </Service>
</Server>
EOF
```

### SARA Server
```bash
# build & deploy
cd ~/SARA-server
mvn clean package -DskipTests
sudo -u tomcat8 cp target/SaraServer-*.war /var/lib/tomcat8/webapps/SaraServer.war
# copy some dependencies manually
sudo -u tomcat8 cp ~/.m2/repository/org/postgresql/postgresql/42.1.4/postgresql-42.1.4.jar /var/lib/tomcat8/lib
sudo -u tomcat8 cp ~/.m2/repository/org/apache/geronimo/specs/geronimo-javamail_1.4_spec/1.6/geronimo-javamail_1.4_spec-1.6.jar /var/lib/tomcat8/lib
sudo -u tomcat8 cp ~/repository/org/apache/geronimo/specs/geronimo-activation_1.0.2_spec/1.1/geronimo-activation_1.0.2_spec-1.1.jar /var/lib/tomcat8/lib/
sudo -u tomcat8 cp ~/repository/org/apache/geronimo/javamail/geronimo-javamail_1.4_provider/1.6/geronimo-javamail_1.4_provider-1.6.jar /var/lib/tomcat8/lib
# copy and adjust config
sudo  cp src/main/webapp/META-INF/context.xml /etc/tomcat8/Catalina/localhost/SaraServer.xml
sudo sed -i 's/demo.sara-project.org/'$(hostname)'/' /etc/tomcat8/Catalina/localhost/SaraServer.xml
vim /etc/tomcat8/Catalina/localhost/SaraServer.xml # set email auth pwd
# launch service
sudo service tomcat8 restart
```
