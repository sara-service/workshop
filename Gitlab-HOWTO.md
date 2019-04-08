# Setting up a GitLab in bwCloud

## Initial fixes
* `ssh ubuntu@vm-XXX-XXX.bwcloud.uni-ulm.de`
* `sudo hostname vm-XXX-XXX.bwcloud.uni-ulm.de`
* `sudo locale-gen de_DE.UTF-8 en_US.UTF-8`
* `sudo localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8`

## Install the GitLab Omnibus package
* `wget -O- https://packages.gitlab.com/gitlab/gitlab-ce/gpgkey | sudo apt-key add`
* `sudo apt-key list` and verify that this added key `E15E78F4` (last 8 digits of key hash)
* add the repo
```
sudo sh -c 'cat << EOF > /etc/apt/sources.list.d/gitlab-ce.list
deb https://packages.gitlab.com/gitlab/gitlab-ce/ubuntu/ bionic main
deb-src https://packages.gitlab.com/gitlab/gitlab-ce/ubuntu/ bionic main
EOF'
```
* `sudo apt update`
* install GitLab CE: sudo EXTERNAL_URL=http://$(hostname) apt install gitlab-ce

## Setting admin password 
* visit http://$(hostname)
* enter the password
* hope that nobody else is faster than you

Yes, that's the official method :(

## Setting up SSL with LetsEncrypt

* in `/etc/gitlab/gitlab.rb`, set variables as follows (unless you need other things, that's actually the entire config!)
```
external_url 'http://gitlabdomain' # note http://; no SSL yet!
nginx['custom_gitlab_server_config'] = "location ^~ /.well-known { root /var/www/letsencrypt; }"
```

* reconfigure and restart gitlab: `sudo gitlab-ctl reconfigure && sudo gitlab-ctl restart`
* get SSL certificate: `sudo letsencrypt certonly --webroot -w /var/www/letsencrypt -d gitlabdomain`
* again in `/etc/gitlab/gitlab.rb`, set (again, that's usually the entire config)
```
external_url 'https://gitlabdomain' # note https:// now!
nginx['redirect_http_to_https'] = true
nginx['ssl_certificate'] = "/etc/letsencrypt/live/gitlabdomain/fullchain.pem"
nginx['ssl_certificate_key'] = "/etc/letsencrypt/live/gitlabdomain/privkey.pem"
nginx['custom_gitlab_server_config'] = "location ^~ /.well-known { root /var/www/letsencrypt; }"
```

* highly recommended: go to https://mozilla.github.io/server-side-tls/ssl-config-generator/ and add something like this as well:
```
nginx['ssl_protocols'] = "TLSv1.2";
nginx['ssl_ciphers'] = "CIPHER-LIST:GOES-HERE"
nginx['ssl_prefer_server_ciphers'] = "on"
nginx['hsts_max_age'] = 15768000
```
* reconfigure and restart gitlab again (sudo gitlab-ctl reconfigure && sudo gitlab-ctl restart)
* visit `https://www.ssllabs.com/ssltest/analyze.html?d=$(hostname)` and verify you get an A or A+
* set up periodic renewal: create /etc/cron.d/letsencrypt containing

```
15 3 * * * root /usr/bin/letsencrypt renew && gitlab-ctl restart nginx
```
occasionally re-check the config generator
