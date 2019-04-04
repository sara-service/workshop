# Install & Configure DSpace6 for SARA using bwCloud

## Intro

This manual provides a step-by-step setup for a fully configured instance of DSpace6 server. 
The final instance can be used as institutional repository to receive automated deposits from SARA Service via swordv2.

Further reading:
https://wiki.duraspace.org/display/DSDOC6x/DSpace+6.x+Documentation

About SARA:
https://sara-service.org

In case of questions please contact:
* Stefan Kombrink, Ulm University, Germany / e-mail: stefan.kombrink[at]uni-ulm.de
* Volodymyr Kushnarenko, Ulm University, Germany / e-mail: volodymyr.kushnarenko[at]uni-ulm.de
* Franziska Rapp, Ulm University, Germany / e-mail: franziska.rapp[at]uni-ulm.de

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
git clone https://github.com/sara-service/workshop.git
```

## Installation

```bash
sudo apt-mark hold openjdk-11-jre-headless # do NOT install java 11
sudo apt-get update && sudo apt-get -y upgrade
sudo apt-get -y install openjdk-8-jdk maven ant postgresql postgresql-contrib curl wget haveged ruby-compass ruby-sass
```

### Postgres
```bash
sudo systemctl start postgresql
sudo groupadd dspace
sudo useradd -m -g dspace dspace
sudo -u postgres createuser --no-superuser dspace
sudo -u postgres psql -c "ALTER USER dspace WITH PASSWORD 'dspace';"
sudo -u postgres createdb --owner=dspace --encoding=UNICODE dspace
sudo -u postgres psql dspace -c "CREATE EXTENSION pgcrypto;"
```

### Tomcat

```bash
wget http://archive.apache.org/dist/tomcat/tomcat-8/v8.5.32/bin/apache-tomcat-8.5.32.tar.gz -O /tmp/tomcat.tgz
sudo mkdir /opt/tomcat
sudo tar xzvf /tmp/tomcat.tgz -C /opt/tomcat --strip-components=1
sudo cp ~/workshop/DSpace/config/tomcat/tomcat.service /etc/systemd/system/tomcat.service
sudo cp ~/workshop/DSpace/config/tomcat/server.xml /opt/tomcat/conf/server.xml
sudo chown -R dspace.dspace /opt/tomcat
sudo systemctl daemon-reload
sudo systemctl start tomcat
```

Now you should be able to find your tomcat running under http://vm-XXX-XXX.bwcloud.uni-ulm.de:8080

### DSpace
```bash
# install mirage2 deps locally
sudo -H -u dspace sh -c 'wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash'
sudo -H -u dspace bash -c 'export NVM_DIR="$HOME/.nvm" && source "$NVM_DIR/nvm.sh" && nvm install v8'
sudo -H -u dspace bash -c 'export NVM_DIR="$HOME/.nvm" && source "$NVM_DIR/nvm.sh" && npm install -g bower grunt grunt-cli'
```
```bash
wget https://github.com/DSpace/DSpace/releases/download/dspace-6.3/dspace-6.3-src-release.tar.gz -O /tmp/dspace-src.tgz
mkdir -p /tmp/dspace-src
tar -xzvf /tmp/dspace-src.tgz -C /tmp/dspace-src --strip-components=1
sudo chown -R dspace:dspace /tmp/dspace-src 
```
```bash
sudo mkdir /dspace
sudo chown dspace /dspace
sudo chgrp dspace /dspace
```
```bash
# NOTE needs sudo interactive or else build fails for Mirage2(xmlui)
sudo -H -u dspace bash -c 'export GEM_HOME=/var/lib/gems/2.5.0 && export GEM_PATH=/var/lib/gems/2.5.0 && export NVM_DIR="$HOME/.nvm" && source "$NVM_DIR/nvm.sh" && cd /tmp/dspace-src && mvn -e clean package -Dmirage2.on=true -Dmirage2.deps.included=false'
sudo -H -u dspace -- sh -c 'cd /tmp/dspace-src/dspace/target/dspace-installer; ant fresh_install'
```
```bash
# export admins email = it is used by the script to create the bibliography, too
export ADMIN_EMAIL="dspace-admin@notexisting.com"
# Create dspace admin
sudo -u dspace /dspace/bin/dspace create-administrator -e $ADMIN_EMAIL -f "Super" -l "User" -p "iamthebest" -c en
```

### Apply presets

```bash
# Enable Mirage2 Themes
cat ~/workshop/DSpace/config/xmlui.xconf             | sudo -u dspace sh -c 'cat > /dspace/config/xmlui.xconf'
# Apply customized item submission form
cat ~/workshop/DSpace/config/item-submission.xml     | sudo -u dspace sh -c 'cat > /dspace/config/item-submission.xml'
cat ~/workshop/DSpace/config/input-forms.xml         | sudo -u dspace sh -c 'cat > /dspace/config/input-forms.xml'
# Custom item view
cat ~/workshop/DSpace/config/xmlui/item-view.xsl     | sudo -u dspace sh -c 'cat > /dspace/webapps/xmlui/themes/Mirage2/xsl/aspect/artifactbrowser/item-view.xsl'
# Custom messages
cat ~/workshop/DSpace/config/xmlui/messages.xml      | sudo -u dspace sh -c 'cat > /dspace/webapps/xmlui/i18n/messages.xml'
cat ~/workshop/DSpace/config/xmlui/messages_de.xml   | sudo -u dspace sh -c 'cat > /dspace/webapps/xmlui/i18n/messages_de.xml'
# Custom landing page
cat ~/workshop/DSpace/config/xmlui/news-xmlui.xml    | sudo -u dspace sh -c 'cat > /dspace/config/news-xmlui.xml'
# Custom thumbnails
cat ~/workshop/DSpace/config/xmlui/Logo_SARA_RGB.png | sudo -u dspace sh -c 'cat > /dspace/webapps/xmlui/themes/Mirage2/images/Logo_SARA_RGB.png'
# Custom icons
cat ~/workshop/DSpace/config/xmlui/arrow.png         | sudo -u dspace sh -c 'cat > /dspace/webapps/xmlui/themes/Mirage2/images/arrow.png'
# Copy email templates
sudo cp ~/workshop/DSpace/config/emails/* /dspace/config/emails/
sudo chown -R dspace /dspace/config/emails
sudo chgrp -R dspace /dspace/config/emails
# Apply custom local configurations
cat ~/workshop/DSpace/config/local.cfg | sed 's/devel-dspace.sara-service.org/'$(hostname)'/g' | sudo -u dspace tee /dspace/config/local.cfg
# Apply default deposit license
cat ~/workshop/DSpace/config/default.license | sudo -u dspace tee /dspace/config/default.license
```
```bash
# Copy all webapps from dspace to tomcat
sudo rsync -a -v -z --delete --force /dspace/webapps/ /opt/tomcat/webapps
```
```bash
sudo systemctl restart tomcat
sudo systemctl enable postgresql
sudo systemctl enable tomcat
```

### Test your instance
Open the start page of your DSpace server: http://vm-XXX-XXX.bwcloud.uni-ulm.de:8080/xmlui
You should be able to login with your admin account.

## Configuration

### Create an initial configuration

First we will create a sample bibliography:

        Faculty of Education
                -> Publications
                -> Research Data
        Faculty of Science and Technology
                -> Publications
                -> Research Data
                
```bash
sudo -u dspace /dspace/bin/dspace structure-builder -f ~/workshop/DSpace/config/DSpace_Import_Structure.xml -o /tmp/DSpace_Export_Structure.xml -e "$ADMIN_EMAIL"
```

Now we will create a service user dedicated to SARA<sup>*</sup>
```bash
sudo /dspace/bin/dspace user --add --email project-sara@uni-konstanz.de --password SaraTest --givenname SARA --surname ServiceUser
```

...and two demo users
```bash
sudo /dspace/bin/dspace user --add --email demo-user@sara-service.org --password SaraTest --givenname Demo --surname User
sudo /dspace/bin/dspace user --add --email demo-user-noaccess@sara-service.org --password SaraTest --givenname Demo --surname Loser
```

After that, we need to create groups and configure permissions. You will need to login as admin using the DSpace UI: 
* create a group called `SARA User` and add `project-sara@uni-konstanz.de`
* create a group called `DSpace User` and add `demo-user@sara-service.org`
* for the two `Research Data` collections allow submissions for both `DSpace User` and `SARA User`
* for one `Publication` collection allow submissions for `DSpace User` only
* for the other `Publication` collection allow submissions for `SARA User` only

<sup>*</sup>this is the dedicated SARA Service user and needs to have permissions to submit to any collection a SARA user has access to! You can even use a non-existing email address since you are admin.

### Validate Swordv2 functionality (HTTP)

Now we check whether the Sword Interface is configured properly and a valid ServiceDocument is being delivered.
We distinguish three cases
*USER1* is registered and has access to at least one collection
*USER2* is registered but has no access to any collection
*USER3* is not registered at all

```bash
DSPACE_SERVER="$(hostname):8080"

SARA_USER="project-sara@uni-konstanz.de"
SARA_PWD="SaraTest"
USER1="demo-user@sara-service.org" # set existing SARA User
USER2="demo-user-noaccess@sara-service.org" # set existing user without any permissions
USER3="daniel.duesentrieb@entenhausen.de" # set nonexisting user

curl -H "on-behalf-of: $USER1" -i $DSPACE_SERVER/swordv2/servicedocument --user "$SARA_USER:$SARA_PWD"  # => downloads TermsOfServices for all available collections
curl -H "on-behalf-of: $USER2" -i $DSPACE_SERVER/swordv2/servicedocument --user "$SARA_USER:$SARA_PWD"  # => downloads empty service document
curl -H "on-behalf-of: $USER3" -i $DSPACE_SERVER/swordv2/servicedocument --user "$SARA_USER:$SARA_PWD"  # => HTML Error Status 403: Forbidden
```

### Install apache httpd
```bash
sudo apt-get -y install apache2
sudo a2enmod ssl proxy proxy_http proxy_ajp
sudo systemctl restart apache2
```

Now you will see the standard apache index page: http://vm-XXX-XXX.bwcloud.uni-ulm.de

### Install letsencrypt, create and configure SSL cert
```bash
sudo apt -y install python3-certbot-apache
sudo systemctl stop apache2
sudo letsencrypt --authenticator standalone --installer apache --domains $(hostname)
```
Choose `secure redirect` . Now you should be able to access via https only: http://vm-XXX-XXX.bwcloud.uni-ulm.de

### Configure apache httpd
First stop tomcat:
```bash
sudo systemctl stop tomcat
```
Then append the following section to your virtual server config under `/etc/apache2/sites-enabled/000-default-le-ssl.conf` :
```bash
sudo vim /etc/apache2/sites-enabled/000-default-le-ssl.conf
```
```apache
        ProxyPass /xmlui ajp://localhost:8009/xmlui
        ProxyPassReverse /xmlui ajp://localhost:8009/xmlui

        ProxyPass /oai ajp://localhost:8009/oai
        ProxyPassReverse /oai ajp://localhost:8009/oai
        ProxyPass /rest ajp://localhost:8009/rest
        ProxyPassReverse /rest ajp://localhost:8009/rest
        ProxyPass /solr ajp://localhost:8009/solr
        ProxyPassReverse /solr ajp://localhost:8009/solr
        ProxyPass /swordv2 ajp://localhost:8009/swordv2

        ProxyPass / ajp://localhost:8009/xmlui
        ProxyPassReverse / ajp://localhost:8009/xmlui
```
Restart apache:
```bash
sudo systemctl restart apache2
```

### Update DSpace local.cfg

Now you need to remove the local port 8080 and the http in the dspace config:
```bash
sudo sed -i 's#dspace.baseUrl = http://${dspace.hostname}:8080#dspace.baseUrl = https://${dspace.hostname}#' /dspace/config/local.cfg
sudo service tomcat restart
```

### Validate Swordv2 functionality (HTTPS)

Experience shows that many things can break while setting up apache+SSL hence we will repeat the previous checks.

```bash
DSPACE_SERVER="https://$(hostname)"

SARA_USER="project-sara@uni-konstanz.de"
SARA_PWD="SaraTest"
USER1="demo-user@uni-ulm.de" # set existing SARA User
USER2="demo-user-noaccess@sara-service.org" # set existing user without any permissions
USER3="daniel.duesentrieb@uni-entenhausen.de" # set nonexisting user

curl -H "on-behalf-of: $USER1" -i $DSPACE_SERVER/swordv2/servicedocument --user "$SARA_USER:$SARA_PWD"  # => downloads TermsOfServices for all available collections
curl -H "on-behalf-of: $USER2" -i $DSPACE_SERVER/swordv2/servicedocument --user "$SARA_USER:$SARA_PWD"  # => downloads empty service document
curl -H "on-behalf-of: $USER3" -i $DSPACE_SERVER/swordv2/servicedocument --user "$SARA_USER:$SARA_PWD"  # => HTML Error Status 403: Forbidden
```
