# Deploy on Amazon Linux

This app is a static Vite build that talks directly to LibCal from the browser.
There is no local Node API server in the current implementation.

These instructions assume:

- Amazon Linux 2023
- Apache httpd serving the site
- the app will live at `/var/www/bookinggrid`
- the public URL will be either the site root or `/reserve`

## 1. Install System Packages

```bash
sudo dnf update -y
sudo dnf install -y git nodejs npm httpd mod_ssl
```

Enable and start Apache:

```bash
sudo systemctl enable httpd
sudo systemctl start httpd
```

## 2. Clone the Project

```bash
cd /var/www
sudo git clone https://github.com/your-username/bookinggrid.git
sudo chown -R $USER:$USER /var/www/bookinggrid
cd /var/www/bookinggrid
```

## 3. Create Production Environment File

Create `.env.production` in the project root.

Minimum required values:

```dotenv
VITE_LIBCAL_BASE_URL=https://uri.libcal.com/api/1.1
VITE_LIBCAL_OAUTH_URL=https://uri.libcal.com/api/1.1/oauth/token
VITE_LIBCAL_CLIENT_ID=your_client_id
VITE_LIBCAL_CLIENT_SECRET=your_client_secret
```

Optional overrides:

```dotenv
VITE_LOCATION_ID=23510
VITE_LIBCAL_GROUP_ID=49543
VITE_LIBCAL_PUBLIC_URL=https://uri.libcal.com
VITE_LIBCAL_WIDGET_ID=1ebef3548aa2
VITE_LIBRARY_NAME=Carothers Library
```

Notes:

- `VITE_LOCATION_ID` and `VITE_LIBCAL_GROUP_ID` are optional because the app now has working defaults for the current LibCal setup.
- `VITE_ROOM_ITEM_IDS` is no longer needed for room discovery.
- keep `.env.production` out of source control

## 4. Build the App

```bash
cd /var/www/bookinggrid
npm install
npm run build
```

Build output is written to `dist/`.

## 5. Configure Apache

If you want the app at the site root, point Apache at `dist/`.

If you want the app under `/reserve`, add a config like this:

```apache
Alias /reserve /var/www/bookinggrid/dist

<Directory /var/www/bookinggrid/dist>
  Options FollowSymLinks
  AllowOverride None
  Require all granted

  RewriteEngine On
  RewriteBase /reserve/
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule ^ /reserve/index.html [L]
</Directory>
```

If you serve at the root instead, use:

```apache
DocumentRoot /var/www/bookinggrid/dist

<Directory /var/www/bookinggrid/dist>
  Options FollowSymLinks
  AllowOverride None
  Require all granted

  RewriteEngine On
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule ^ /index.html [L]
</Directory>
```

Place the config in a vhost file such as:

```bash
sudo nano /etc/httpd/conf.d/bookinggrid.conf
```

Then validate and reload Apache:

```bash
sudo apachectl configtest
sudo systemctl reload httpd
```

## 6. Set Permissions

```bash
sudo chown -R apache:apache /var/www/bookinggrid/dist
sudo find /var/www/bookinggrid/dist -type d -exec chmod 755 {} \;
sudo find /var/www/bookinggrid/dist -type f -exec chmod 644 {} \;
```

## 7. Updating the Site

From the project directory:

```bash
git pull
npm install
npm run build
sudo systemctl reload httpd
```

## 8. Troubleshooting

Apache status:

```bash
sudo systemctl status httpd
```

Apache error log:

```bash
sudo tail -f /var/log/httpd/error_log
```

Rebuild the app after env changes:

```bash
npm run build
```

Important deployment detail:

- Vite injects `VITE_*` variables at build time.
- changing `.env.production` on the server does nothing until you rebuild with `npm run build`.
