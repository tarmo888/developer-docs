---
description: >-
  These settings are applicable to most Obyte nodes projects that have
  'headless-obyte' or 'ocore' library as dependency.
---

# Configuration

The default settings are in the core library's `conf.js` file, they can be overridden in your project root's `conf.js` and then in `conf.json` in the app data folder. The app data folder is:

* macOS: `~/Library/Application Support/<appname>`
* Linux: `~/.config/<appname>`
* Windows: `%LOCALAPPDATA%\<appname>`

`<appname>` is `name` in your `package.json`, e.g. `headless-obyte`.

## Database location

At some point you'll surely want to have a peek into the database, or even make a modification in it. Default SQLite database file and RocksDB folder are located near the config file, you can find the correct paths for each platform above.

## Testnet network

Most bot repositories have example `.env.testnet` file in their root folder. In order to connect use testnet network, just run `cp .env.testnet .env`. Backup and delete the database if you already ran it on MAINNET. Wallet app for [TESTNET can be downloaded from Obyte.org](https://obyte.org/testnet.html) website.

If you don't have `.env.testnet` example file then just create a `.env` file that contains this row:

```text
testnet=1
```

## Settings

This is the list of some of the settings that the library understands \(your app can add more settings that only your app understands\):

### conf.port

The port to listen on. If you don't want to accept incoming connections at all, set port to `null`, which is the default. If you do want to listen, you will usually have a proxy, such as nginx, accept websocket connections on standard port 443 and forward them to your Obyte daemon that listens on port 6611 on the local interface.

### conf.storage

Storage backend -- mysql or sqlite, the default is sqlite. If sqlite, the database files are stored in the app data folder. If mysql, you need to also initialize the database with [initial .sql file](https://github.com/byteball/ocore/tree/master/initial-db) and set connection params:

{% code title="conf.json" %}
```javascript
{
    "storage": "mysql",
    "database": {
        "host"     : "localhost",
        "user"     : "DATABASE_USER",
        "password" : "DATABASE_PASSWORD",
        "name"     : "DATABASE_NAME"
    }
}
```
{% endcode %}

#### MySQL conf for faster syncing

To lower disk load and increase sync speed, you can optionally disable flushing to disk every transaction, instead doing it once a second. This can be done by setting `innodb_flush_log_at_trx_commit=0` in your MySQL server config file \(my.ini\)

### conf.bLight

Work as light node \(`true`\) or full node \(`false`\). The default is full client. Light node only holds data that is relevant to your wallet, full node sync all the data in public database. Benefit of light node is that you can start using it immediately, but the downside is that you might not get updates about new transactions and confirmations as fast as full node does.

### conf.bServeAsHub

Whether to serve as hub on the Obyte network \(store and forward e2e-encrypted messages for devices that connect to your hub\). The default is `false`.

### conf.myUrl

If your node accepts incoming connections, this is its URL. The node will share this URL with all its outgoing peers so that they can reconnect in any direction in the future. By default the node doesn't share its URL even if it accepts connections.

### conf.bWantNewPeers

Whether your node wants to learn about new peers from its current peers \(`true`, the default\) or not \(`false`\). Set it to `false` to run your node in stealth mode so that only trusted peers can see its IP address \(e.g. if you have online wallets on your server and don't want potential attackers to learn its IP\).

### conf.socksHost, conf.socksPort, and conf.socksLocalDNS

Settings for connecting through optional SOCKS5 proxy. Use them to connect through TOR and hide your IP address from peers even when making outgoing connections. This is useful and highly recommended when you are running an online wallet on your server and want to make it harder for potential attackers to learn the IP address of the target to attack. Set `socksLocalDNS` to `false` to route DNS queries through TOR as well.

## Email notifications

Most bots out there expect user's machine to have UNIX sendmail and by default `sendmail` function in `mail` module will try to use that, but it is possible to configure your node to use SMTP relay. This way you could use Gmail or Sendmail SMTP server or even something like Mailtrap.io \(excellent for testing purposes if you don't want the actual email recipient to receive your test messages\). This is how the configuration would look:

{% code title="conf.json" %}
```javascript
{
    "smtpTransport": "relay",
    "smtpRelay": "smtp.mailtrap.io",
    "smtpUser": "MAILTRAP_INBOX_USER",
    "smtpPassword": "MAILTRAP_INBOX_PASSWORD",
    "smtpSsl": false,
    "smtpPort": 2525,
    "admin_email": "admin@example.com",
    "from_email": "bot@example.com"
}
```
{% endcode %}

`smtpTransport` can take one of three values:

* `local`: send email using locally installed `sendmail`. Normally, `sendmail` is not installed by default and when installed, it needs to be properly configured to actually send emails. If you choose this option, no other conf settings are required for email. This is the default option.
* `direct`: send email by connecting directly to the recipient's SMTP server. This option is not recommended.
* `relay`: send email through a relay server, like most email apps do. You need to also configure the server's host `smtpRelay`, its port `smtpPort` if it differs from the default port 25, and `smtpUser` and `smtpPassword` for authentication to the server.

## Accepting incoming connections

Obyte network works over secure WebSocket protocol wss://. To accept incoming connections, you'll need a valid TLS certificate \(you can get a free one from [letsencrypt.org](https://letsencrypt.org)\) and a domain name \(you can get a free domain from [Freenom](http://www.freenom.com/)\). Then you accept connections on standard port 443 and proxy them to your locally running Obyte daemon.

This is an example configuration for nginx to accept websocket connections at wss://obyte.one/bb and forward them to locally running daemon that listens on port 6611:

If your server doesn't support IPv6, comment or delete the two lines containing \[::\] or nginx won't start

```text
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    listen 443 ssl;
    listen [::]:443 ssl;
    ssl_certificate "/etc/letsencrypt/live/obyte.one/fullchain.pem";
    ssl_certificate_key "/etc/letsencrypt/live/obyte.one/privkey.pem";

    if ($host != "obyte.one") {
        rewrite ^(.*)$ https://obyte.one$1 permanent;
    }
    if ($https != "on") {
        rewrite ^(.*)$ https://obyte.one$1 permanent;
    }

    location = /bb {
        proxy_pass http://localhost:6611;
        proxy_http_version 1.1;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    root /var/www/html;
    server_name _;
}
```

By default Node limits itself to 1.76GB the RAM it uses. If you accept incoming connections, you will likely reach this limit and get this error after some time:

```text
FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory
1: node::Abort() [node]
...
...
12: 0x3c0f805c7567
Out of memory
```

To prevent this, increase the RAM limit by adding `--max_old_space_size=<size>` to the launch command where size is the amount in MB you want to allocate.

For example `--max-old-space-size=4096`, if your server has at least 4GB available.

## Checking if the node is running

### Sending the notification to admin when node stops

Many bots have a file named [check\_daemon.js](https://github.com/byteball/bot-example/blob/master/check_daemon.js) included in the repository, which looks like this

```javascript
const check_daemon = require('ocore/check_daemon.js');
check_daemon.checkDaemonAndNotify('node start.js');
```

This can be executed with crontab with the command mentioned in [crontab.txt](https://github.com/byteball/bot-example/blob/master/crontab.txt) and when the email is properly configured on the server and the node crashes, this script will notify you that the node went down.

If you are executing your node with `node start.js --max-old-space-size=4096` command then you should also change the `checkDaemonAndNotify` parameter.

### Restarting the node automatically when it stops

If you are running a wallet-less node \(Hub, Relay or Explorer\) or password-less headless wallet \(`conf.bNoPassphrase=true`\) then you can restart the node automatically when it stops. In order to do that, you could use `checkDaemonAndRestart` function, instead of `checkDaemonAndNotify`.

```javascript
const check_daemon = require('ocore/check_daemon.js');
check_daemon.checkDaemonAndRestart('node start.js', 'node start.js 1>log 2>>err');
```

The first parameter in `checkDaemonAndRestart` is the process that gets searched from `ps x` command response, the second parameter is a command that will get executed when node has stopped.

Above example is for wallet-less node, which will log the `stdout` and `stderr`  \(&gt;&gt; for appending\) to file \([headless wallet has logging built-in](tutorials-for-newcomers/setting-up-headless-wallet.md#monitoring-logs)\). If the command uses parameters like `--max-old-space-size=4096` then these should be added to both parameters, but output directing \(`1>log` and `2>>err`\) should not be added to first parameter.

If you are using NVM to manage your Node.js versions then you might need to add `PATH` and `LD_LIBRARY_PATH` to your crontab configuration as well, otherwise crontab might not be able to restart the node.

