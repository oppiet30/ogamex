# Install OGameX without Docker

This guide runs OGameX directly on Linux with PHP, MariaDB, and Node.js. Fedora commands are provided; on other distributions install equivalent packages and adjust service names, PHP configuration paths, and the web-server user.

Run project commands from the repository root as your normal deployment user unless `sudo` is shown. These instructions follow this branch's `composer.json`, `package-lock.json`, configuration files, and startup script rather than assuming that the Docker environment exists.

## 1. Install the requirements

You need:

- PHP **8.5.x** for both the command line and PHP-FPM (the project requires `^8.5`, with Laravel 13).
- Composer 2.
- MariaDB (the bundled environment uses MariaDB 11.3.2) or a compatible MySQL server.
- Node.js **22.12 or newer** and npm for building the locked Vite dependencies.
- Git, unzip, a C compiler/linker, Rust, and Cargo. Rust is the default battle engine; see the PHP alternative below.
- PHP extensions: Ctype, cURL, DOM/XML, Fileinfo, Filter, Hash, Mbstring, OpenSSL, PCRE, PDO, PDO MySQL, Session, Tokenizer, ZIP, GD, EXIF, and FFI. PCNTL is needed for reliable queue worker signals/timeouts; OPcache is recommended for PHP-FPM.

On Fedora:

```bash
sudo dnf install git unzip composer mariadb-server mariadb \
    php-cli php-fpm php-common php-mysqlnd php-mbstring php-xml \
    php-gd php-zip php-ffi php-process php-opcache \
    nodejs npm rust cargo gcc make policycoreutils-python-utils
sudo systemctl enable --now mariadb
php -v
php -m
composer --version
node --version
npm --version
cargo --version
```

Package availability depends on your Fedora release and enabled repositories. Check the versions before continuing: if the installed PHP or Node.js is too old, install a compatible version first. Do not use Composer's `--ignore-platform-reqs`. PHP-FPM must use the same compatible PHP version and extensions as the CLI.

## 2. Download the project

```bash
git clone https://github.com/oppiet30/ogamex.git
cd ogamex
cp .env.example .env
chmod 600 .env
```

For a repeatable deployment, check out a reviewed release or commit compatible with this guide. Keep `.env` out of Git.

## 3. Create a database and application user

Open the local database administration shell:

```bash
sudo mariadb
```

Run this SQL, replacing the example password with a unique password:

```sql
CREATE DATABASE ogamex CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'ogamex'@'127.0.0.1' IDENTIFIED BY 'REPLACE_WITH_A_UNIQUE_PASSWORD';
GRANT ALL PRIVILEGES ON ogamex.* TO 'ogamex'@'127.0.0.1';
EXIT;
```

The application connects over TCP to `127.0.0.1`, matching the user above. Test it without putting the password in your shell history:

```bash
mariadb -h 127.0.0.1 -u ogamex -p ogamex
```

## 4. Configure `.env`

Edit the existing entries in `.env` and add missing ones. For a local installation on this computer:

```dotenv
APP_NAME=OGameX
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost:8000
DEBUGBAR_ENABLED=false

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=ogamex
DB_USERNAME=ogamex
DB_PASSWORD="REPLACE_WITH_A_UNIQUE_PASSWORD"

CACHE_STORE=file
SESSION_DRIVER=database
QUEUE_CONNECTION=database
BROADCAST_CONNECTION=reverb
MAIL_MAILER=log

REVERB_APP_ID=ogamex
REVERB_APP_KEY=REPLACE_WITH_A_RANDOM_KEY
REVERB_APP_SECRET=REPLACE_WITH_A_RANDOM_SECRET
REVERB_HOST=localhost
REVERB_PORT=8090
REVERB_SCHEME=http
REVERB_SERVER_HOST=127.0.0.1
REVERB_SERVER_PORT=8090
```

Generate separate random values for the Reverb key and secret, for example by running this twice:

```bash
openssl rand -hex 32
```

`REVERB_APP_KEY` is sent to the browser; `REVERB_APP_SECRET` must remain private. Redis is not required for these file-cache/database-queue settings with Reverb scaling disabled.

The copied examples contain older names such as `CACHE_DRIVER`, `MAIL_DRIVER`, and, in the production example, `QUEUE_DRIVER` and `BROADCAST_DRIVER`. The current configuration reads **`CACHE_STORE`, `MAIL_MAILER`, `QUEUE_CONNECTION`, and `BROADCAST_CONNECTION`**. Use those names explicitly. Also replace the Docker-only database hostname `ogamex-db`.

`MAIL_MAILER=log` writes emails to the application log; it does not deliver them. For real email delivery, set `MAIL_MAILER=smtp` and configure `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_ENCRYPTION`, and `MAIL_FROM_ADDRESS` for your mail provider.

## 5. Install PHP dependencies and initialize Laravel

```bash
composer install
composer check-platform-reqs
php artisan key:generate
mkdir -p storage/framework/cache/data storage/framework/sessions \
    storage/framework/views storage/logs storage/rust-libs bootstrap/cache
php artisan migrate
php artisan storage:link
```

Use `composer install --no-dev --optimize-autoloader` instead for production. Always use the committed lock file; do not run `composer update` as an installation step.

Migrations create the application tables, including sessions and queued jobs. There is no database-seeding step required by the bundled startup script. Generate `APP_KEY` once for a new installation and preserve it across upgrades.

## 6. Build the battle engine

The default server setting selects the Rust engine, so build its native libraries:

```bash
sh rust/compile.sh
ls -l storage/rust-libs/libbattle_engine_ffi.so
```

Build on the target machine, or an environment matching its architecture and system libraries. Do not copy an x86-64 `.so` onto a Raspberry Pi running ARM Linux.

Enable FFI in both the CLI and PHP-FPM PHP configurations used by this application:

```ini
ffi.enable=true
```

Find CLI configuration files with `php --ini`; check the PHP-FPM configuration separately. Restart PHP-FPM after changes:

```bash
sudo systemctl restart php-fpm
php -r 'var_dump(extension_loaded("FFI"), ini_get("ffi.enable"));'
```

The native library must be readable by the web-server and worker users. If you deliberately do not want Rust/FFI, register the first account and select **PHP** as the battle engine in the admin server settings before running any battles. The default is Rust; skipping compilation alone does not select the PHP implementation.

## 7. Build browser assets

```bash
npm ci
npm run build
```

This creates `public/build/manifest.json` and the CSS/JavaScript bundles. Node.js is needed for builds, not for serving the compiled website. Rebuild after frontend updates. For frontend development you may use `npm run dev` instead while the Vite process remains running.

## 8. Start a local installation

In four separate terminals, each in the project root, run:

```bash
# Terminal 1: website
php artisan serve --host=127.0.0.1 --port=8000
```

```bash
# Terminal 2: queued jobs and broadcasts
php artisan queue:work --sleep=1 --tries=3 --timeout=60
```

```bash
# Terminal 3: scheduled tasks (highscores, cleanup, regeneration)
php artisan schedule:work
```

```bash
# Terminal 4: real-time chat WebSocket server
php artisan reverb:start --host=127.0.0.1 --port=8090
```

Visit <http://localhost:8000> and create your account. **The first registered account becomes the administrator.** Closing these terminals stops their processes. Laravel's development server is for local testing; use PHP-FPM and a web server for a persistent deployment.

For LAN testing, bind the website and Reverb to the server's LAN address and set `APP_URL` and `REVERB_HOST` to an address reachable from the client browser. `localhost` in a remote browser means that client's computer. Allow only the required ports through your firewall.

## 9. Persistent deployment with Apache and PHP-FPM on Fedora

The following assumes the checked-out project is installed at `/var/www/ogamex`, with assets, dependencies, `.env`, and native libraries already prepared there. Keep source files owned by your deployment user. Fedora's default PHP-FPM pool runs as `apache`; verify `/etc/php-fpm.d/www.conf` if yours differs.

```bash
sudo dnf install httpd mod_ssl
sudo chgrp apache /var/www/ogamex/.env
sudo chmod 640 /var/www/ogamex/.env
sudo chown -R apache:apache /var/www/ogamex/storage /var/www/ogamex/bootstrap/cache
sudo chmod -R u+rwX,g+rwX /var/www/ogamex/storage /var/www/ogamex/bootstrap/cache
sudo semanage fcontext -a -t httpd_sys_content_t '/var/www/ogamex(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/ogamex/storage(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/ogamex/bootstrap/cache(/.*)?'
sudo restorecon -Rv /var/www/ogamex
sudo setsebool -P httpd_can_network_connect_db on
sudo setsebool -P httpd_can_network_connect on
sudo systemctl enable --now php-fpm httpd
```

The network booleans allow database connections and outbound connections such as Reverb/SMTP. If a file-context rule already exists, modify it with `semanage fcontext -m` instead of adding it again. Do not disable SELinux or use `chmod 777`. Ensure Apache can traverse parent directories and read the source and built assets.

Configure an Apache virtual host whose **DocumentRoot is `/var/www/ogamex/public`**, never the repository root:

```apache
<VirtualHost *:80>
    ServerName game.example.com
    DocumentRoot /var/www/ogamex/public

    <Directory /var/www/ogamex/public>
        AllowOverride All
        Require all granted
        Options -Indexes +FollowSymLinks
    </Directory>
</VirtualHost>
```

Save it as `/etc/httpd/conf.d/ogamex.conf`. Fedora's PHP-FPM integration and Apache rewrite module must be enabled; `public/.htaccess` handles Laravel routing. Validate and reload:

```bash
sudo apachectl configtest
sudo systemctl reload httpd
```

For public hosting, configure a valid TLS certificate and HTTPS virtual host before setting:

```dotenv
APP_ENV=production
APP_DEBUG=false
APP_URL=https://game.example.com
```

OGameX forces HTTPS URLs in production. Use `APP_ENV=local` only for the earlier local HTTP setup.

### HTTPS WebSocket proxy

For HTTPS pages, proxy Reverb through a TLS virtual host. One option is a dedicated hostname, `ws.example.com`, with its own DNS record and valid certificate. Set:

```dotenv
REVERB_HOST=ws.example.com
REVERB_PORT=443
REVERB_SCHEME=https
REVERB_SERVER_HOST=127.0.0.1
REVERB_SERVER_PORT=8090
```

Inside the Apache HTTPS virtual host for `ws.example.com`, configure:

```apache
ProxyPreserveHost On
ProxyPass /app/ http://127.0.0.1:8090/app/ upgrade=websocket
ProxyPassReverse /app/ http://127.0.0.1:8090/app/
ProxyPass /apps/ http://127.0.0.1:8090/apps/
ProxyPassReverse /apps/ http://127.0.0.1:8090/apps/
```

This requires Apache's proxy, proxy_http, and SSL modules and an Apache version supporting `upgrade=websocket` (2.4.47+). `/app/` carries browser WebSockets; `/apps/` carries server-side broadcast API requests. The application server must resolve and reach `ws.example.com` over HTTPS too. Keep internal port 8090 on loopback.

If firewalld is active, allow HTTP/HTTPS in the zone serving your network, for example:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

These commands use the default zone; add `--zone=YOUR_ZONE` when needed. MariaDB and internal Reverb ports need not be exposed publicly.

### Keep background processes running with systemd

Create `/etc/systemd/system/ogamex-queue.service`:

```ini
[Unit]
Description=OGameX queue worker
After=network.target mariadb.service

[Service]
User=apache
Group=apache
WorkingDirectory=/var/www/ogamex
ExecStart=/usr/bin/php artisan queue:work --sleep=1 --tries=3 --timeout=60
Restart=always
RestartSec=5
TimeoutStopSec=90

[Install]
WantedBy=multi-user.target
```

Create `/etc/systemd/system/ogamex-reverb.service` with the same content, changing `Description` and `ExecStart` to:

```ini
Description=OGameX Reverb server
ExecStart=/usr/bin/php artisan reverb:start --host=127.0.0.1 --port=8090
```

For the scheduler, create `/etc/systemd/system/ogamex-scheduler.service`:

```ini
[Unit]
Description=OGameX scheduler
After=network.target mariadb.service

[Service]
User=apache
Group=apache
WorkingDirectory=/var/www/ogamex
ExecStart=/usr/bin/php artisan schedule:work
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Verify `/usr/bin/php` is your PHP 8.5 executable, then enable the services:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ogamex-queue ogamex-reverb ogamex-scheduler
sudo systemctl status ogamex-queue ogamex-reverb ogamex-scheduler
```

As an alternative to the scheduler service, run `php artisan schedule:run` once per minute using cron as the application user, with an absolute path and project working directory. Use one scheduler method, not both.

### Cache considerations

After configuration changes, clear stale caches as the application user:

```bash
cd /var/www/ogamex
sudo -u apache /usr/bin/php artisan optimize:clear
sudo -u apache /usr/bin/php artisan route:cache
sudo -u apache /usr/bin/php artisan view:cache
```

**Do not enable `config:cache` for this setup yet.** The current chat layout reads `REVERB_*` directly through `env()` in a Blade view. Laravel configuration caching prevents `.env` from being loaded for those reads, which can break chat. This differs from the bundled production startup script. Keep configuration uncached unless those view reads are migrated to `config()` or the variables are supplied to every runtime through the process environment.

## 10. Verify and troubleshoot

```bash
php artisan migrate:status
php artisan schedule:list
php artisan queue:failed
```

Check that registration/login work, assets load, highscores update on schedule, and chat messages arrive between two signed-in users. Test a battle after verifying the selected engine. For persistent deployments:

```bash
sudo journalctl -u ogamex-queue -u ogamex-reverb -u ogamex-scheduler -n 100
sudo journalctl -u php-fpm -u httpd -n 100
```

Application logs are normally under `storage/logs/`.

| Symptom | Check |
| --- | --- |
| Composer refuses installation | PHP version, required extensions, and `composer check-platform-reqs`; do not bypass requirements. |
| Database connection fails | Replace `ogamex-db` with `127.0.0.1`; check database user/password, MariaDB service, and SELinux database access. |
| HTTP 500 or permission denied | Application logs, `.env` readability, writable `storage`/`bootstrap/cache`, and SELinux labels. |
| Missing Vite manifest or broken assets | Run `npm ci` and `npm run build`; remove a stale `public/hot` file if the Vite development server is no longer running. |
| Battle fails loading FFI/library | Enable FFI in the executing PHP runtime, compile for the host, and verify library readability; alternatively explicitly select PHP in server settings. |
| Chat fails | Queue and Reverb processes, browser WebSocket errors, shared Reverb credentials, public hostname/TLS/proxy, and the configuration-cache caveat above. |
| Scheduled updates do not happen | Scheduler service or cron, `schedule:list`, application logs, and database connectivity. |
| HTTPS redirects on a local HTTP setup | Use `APP_ENV=local`, set the correct `APP_URL`, and clear cached configuration. |

To grant administrator access manually:

```bash
php artisan ogamex:admin:assign-role YOUR_USERNAME
```

## 11. Upgrade an existing manual installation

Back up the database, `.env` (including `APP_KEY`), and persistent storage first. Record the current commit so you know what version you are upgrading from. Stop queue, Reverb, and scheduler services and enable maintenance mode before changing files:

```bash
cd /var/www/ogamex
sudo -u apache /usr/bin/php artisan down
sudo systemctl stop ogamex-queue ogamex-reverb ogamex-scheduler
git pull --ff-only origin main
composer install --no-dev --optimize-autoloader
npm ci
npm run build
sh rust/compile.sh
sudo -u apache /usr/bin/php artisan migrate --force
sudo -u apache /usr/bin/php artisan optimize:clear
sudo -u apache /usr/bin/php artisan route:cache
sudo -u apache /usr/bin/php artisan view:cache
sudo systemctl restart php-fpm
sudo systemctl start ogamex-queue ogamex-reverb ogamex-scheduler
sudo -u apache /usr/bin/php artisan up
```

Run Git/Composer/npm/Rust commands as the deployment user, with appropriate group access to existing build outputs. Restore ownership and SELinux labels as in section 9 if needed. For a pinned deployment, check out the reviewed release/commit instead of pulling `main`. Check service logs and repeat the functional checks after upgrading. Do not use `migrate:fresh` or regenerate `APP_KEY` on an existing installation. Database migrations may require restoring a backup rather than simply checking out an older commit to recover.
