# GitLab

## Step 1: Set up a server instance with 4GB RAM and Ubuntu 16.04

1. Log into vultr and click the deploy button in the Servers tab
2. If you haven't added an SSH key to your Vultr account, skip down to
   SSH Keys, and add one, and come back to the Deploy page
3. Choose a server location (the closest one will suffice)
4. Choose Ubuntu 16.04 for the Server Type
5. Choose the server with 4096MB (4GB) memory ($20/month)
6. Choose an SSH key
7. Enter a hostname. `gitlab` is a good one.
8. Click `Deploy Now` and wait
9. Once it's installed, grab the IP address

For info on why GitLab strongly recommends 4GB, see
[GitLab's requirements page](https://docs.gitlab.com/ce/install/requirements.html#memory).

## Step 2: Create a DNS record pointing your domain to your server instance

Add a DNS record from your domain to the IP address from the previous step.

- Type: `A`
- Name: Your domain name
- Value: Your new server's IP address

## Step 3: Install GitLab CE

SSH into your new server. Replace `SERVER_IP` with the IP address of your server.

``` bash
ssh root@SERVER_IP
```

Add the package repository:

``` bash
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
```

Install GitLab. Use the `http` version of your URL. We'll change it to `https` later.
Replace `gitlab.example.com` with your server's domain.

``` bash
sudo EXTERNAL_URL="http://gitlab.example.com" apt install gitlab-ce
```

For more info, see the
[GitLab CE OmniBus docs](https://about.gitlab.com/installation/#ubuntu?version=ce).

## Step 4: Get an SSL certificate with Certbot

This is based on the
[guide from DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-secure-gitlab-with-let-s-encrypt-on-ubuntu-16-04):

``` bash
add-apt-repository -y ppa:certbot/certbot
apt update
apt install -y certbot
```

To perform verification, LetsEncrypt needs to create a file and
verify it by accessing it remotely. The file will be created in
`/var/www/letsencrypt` and accessed from inside the `/.well-known`
path in the server. `nginx`

Create the `/var/www/letsencrypt` directory:

``` bash
mkdir -p /var/www/letsencrypt
```

Add this to the end of `/etc/gitlab/gitlab.rb`:

``` ruby

# Additional configuration
nginx['custom_gitlab_server_config'] = 'location ^~ /.well-known { root /var/www/letsencrypt; }'
```

Apply the GitLab configuration and restart GitLab's services, including nginx:

``` bash
gitlab-ctl reconfigure
```

Run certbot (replace `gitlab.example.com` with your domain):

``` bash
certbot certonly --webroot --webroot-path=/var/www/letsencrypt -d gitlab.example.com
```

# Step 5: Configure GitLab to use https

Make an edit and an addition to your `/etc/gitlab/gitlab.rb`.

In `/etc/gitlab/gitlab.rb`, change the protocol of `external_url` from `http` to `https`. That is, replace this:

``` ruby
external_url 'http://gitlab.example.com'
```

With:

``` ruby
external_url 'https://gitlab.example.com'
```

Add this to the end of `/etc/gitlab/gitlab.rb` (Replace `gitlab.example.com` with your domain):

``` ruby
nginx['custom_nginx_config'] = <<EOF
  server {
    listen *:80;
    listen [::]:80;
    server_name gitlab.example.com;
    location /.well-known {
      root /var/www/letsencrypt;
    }
    location / {
      return 301 https://gitlab.example.com$request_uri;
    }
    access_log /var/log/gitlab/nginx/gitlab.example.com-plain-http.log;
    error_log /var/log/gitlab/nginx/gitlab.example.com-plain-http-error.log;
  }
EOF
nginx['ssl_certificate'] = "/etc/letsencrypt/live/gitlab.example.com/fullchain.pem"
nginx['ssl_certificate_key'] = "/etc/letsencrypt/live/gitlab.example.com/privkey.pem"
```

Double-check that you replaced all occurences of gitlab.example.com with your domain
in the above step.

Configure the firewall to allow HTTPS traffic:

``` bash
sudo ufw allow https
```

Reconfigure GitLab:

``` bash
gitlab-ctl reconfigure
```

Check that everything is running:

``` bash
gitlab-ctl status
```

Check that your LetsEncrypt certificate can renew:

``` bash
certbot renew --dry-run
```

## Step 6: Use GitLab's interface to set a password

Go to your server in your browser. If everything worked, it should ask you for
a password. Then login with the username `root` and your password!

## Step 7: Backups

This configuration meets the system requirements, and is protected by HTTPS, but
until you enable backups, you could lose all your data.

Fortunately, [GitLab has backups built-in](https://docs.gitlab.com/ce/raketasks/backup_restore.html#creating-a-backup-of-the-gitlab-system).

