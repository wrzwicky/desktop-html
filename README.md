# Home Page Desktop

A single-file, zero-dependency home page for your home server that behaves like a computer desktop. Can be managed fully from the browser, no need to edit files on the server.

## Features

- Icons that link to your services
- Frames to group icons
- Tidy up: nudges icon to line them up with their neighbors, and Revert undoes it
- Layout saved on the server so every device sees the same desktop, with a copy kept in each browser as a fallback
- Export and import the layout as a JSON file
- No dependencies, no build step, one HTML file

### AI notes:
 - <a href="https://0rwx.com/"><img src="https://raw.githubusercontent.com/NamtaoProductions/0rwx.com/refs/heads/main/static/AI-X-light.svg" alt="AI:X" width="28" align="absmiddle"> AI made this, with help from a human assistant</a>
 - Generated with Claude AI to contain exactly the features I want, no more or less.
 - I might suggest you do the same rather than use this project.

### Security notes:
 - **No code review!** Claude was trusted.
 - Can run with zero server config. Should be unable to compromise server, no guarantees about the client.
 - AI-generated server config appears to restrict access to the one file needed to store the layout. Unlikely to be able compromise server any further than that one file.
 - To add requirement for a password as well, configure authentication for PUT requests.
 - Anyone who can reach the page can read the layout file. Don't put secrets in icon URLs or text fields.

### Compatibility notes:
 - Can be served from any server software with no config, though layout is only saved to user's browser.
 - Server config required for layout to save onto server. Config is provided below for nginx and Apache 2.4.
 - Page was tested with Chrome and nothing else.

![Screenshot](desktop.jpg)

## Requirements

- A web server. These instructions cover nginx or Apache 2.4 on Debian, Ubuntu, or Raspberry Pi OS
- For server-side saving only: nginx with the WebDAV module (`http_dav_module`) or Apache with `mod_dav` and `mod_dav_fs`

Without server-side saving the page still works. The layout is then stored in each browser separately.

## Install

### 1. Copy the page

Put the HTML file in your web root. This README calls it `desktop.html`; you can name it anything, or use `index.html` to make it your home page.

```
sudo cp desktop.html /var/www/html/desktop.html
```

Open `http://your-server/desktop.html`. You should see the desktop with three example icons. At this point it works, but changes are only remembered in that one browser.

### 2. Enable server-side saving (optional)

This kwill make the layout shared across all your devices. Choose the instructions for your server.

#### 2x. Enable DAV on nginx

<details>

**a. Check that nginx has the DAV module**

```
nginx -V 2>&1 | grep -o http_dav_module
```

If that prints nothing, install it. On Debian and Ubuntu: `sudo apt install nginx-extras`.

**b. Create the data directory and layout file**

The layout file gets its own directory so nginx can write there and nowhere else. Your page and the rest of the web root stay read-only to nginx.

```
sudo mkdir -p /var/www/html/data
echo '{}' | sudo tee /var/www/html/data/layout.json
sudo chown -R www-data:www-data /var/www/html/data
```

If nginx runs as a different user, use that user instead of `www-data`. You can check with `grep '^user' /etc/nginx/nginx.conf`.

**c. Add a location block to your nginx `server` block**

For example in `/etc/nginx/sites-enabled/default`:

```nginx
location = /data/layout.json {
    root /var/www/html;
    dav_methods PUT;
    dav_access user:rw group:rw all:r;
    client_max_body_size 1m;
    add_header Cache-Control "no-store";

    # Anyone can read the layout, but only your LAN can write it
    limit_except GET HEAD {
        allow 192.168.0.0/16;
        allow 10.0.0.0/8;
        allow 172.16.0.0/12;
        allow 127.0.0.1;
        allow ::1;
        allow fe80::/10;
        allow fc00::/7;
        # allow 2001:db8:1234:5678::/64;   # ISP-assigned IPv6 prefix if needed (my router has the same v6 IP inside and out)
        deny all;
    }
}
```

Test the configuration and reload:

```
sudo nginx -t && sudo systemctl reload nginx
```

**d. Point the page at the file**

Near the top of the script in the HTML file, find this line and set it to match the location above:

```js
const SERVER = "data/layout.json";
```

**e. Verify**

Open the page in your browser.
</details>

#### 2x. Enable DAV on Apache 2

<details>

The repository includes a ready-to-install Apache configuration at [`apache2/desktop-html.conf`](apache2/desktop-html.conf). It enables WebDAV only for the data directory, limits it to GET/HEAD/PUT, restricts writes to LAN addresses, and denies requests to other paths in `/data/`. Adjust the allowed LAN ranges and web-root paths in the file to suit your setup.

**a. Enable WebDAV and create its lock database directory**

```
sudo a2enmod dav dav_fs
sudo mkdir -p /var/lib/dav/desktop-html
sudo chown www-data:www-data /var/lib/dav/desktop-html
```

**b. Create the layout file and install the configuration**

```
sudo mkdir -p /var/www/html/data
sudo chown -R www-data:www-data /var/www/html/data
sudo cp desktop-html.conf /etc/apache2/conf-available/desktop-html.conf
sudo a2enconf desktop-html
```

The Apache server needs write access to the data directory and DAV lock directory. On Debian/Ubuntu this is normally `www-data`; check your Apache `User` setting if yours differs. Keep the rest of the web root read-only to Apache.

**c. Test and reload Apache**

```
sudo apachectl configtest && sudo systemctl reload apache2
```


**d. Point the page at the file**

Near the top of the script in the HTML file, find this line and set it to match the location above:

```js
const SERVER = "data/layout.json";
```

**e. Verify**

Open the page in your browser.
</details>

## Using it

 - Left click an icon to open its page.
 - Lift click drag to move things around.
 - Right click for context menus.

### Tuning Tidy up

Tidy up lines up icons whose x (or y) positions are within 40 px of each other. To change that, edit the constant in the script:

```js
const ALIGN = 40;
```

## Troubleshooting

If the taskbar shows **⚠ not synced**, hover over it. The tooltip shows the failing request.

| Message | Meaning and fix |
| --- | --- |
| `PUT 405` or `501` | The server isn't accepting PUT for that path. Check that the `location` path matches the `SERVER` value exactly and that the DAV module is installed |
| `PUT 403` | Either the `limit_except` rule rejected your device, or nginx can't write the file. Run `sudo tail -n 5 /var/log/nginx/error.log`: "access forbidden by rule" means the address in the `allow` list did not match; a filesystem error means to check permissions |
| `PUT 404` or `409` | The `data` directory doesn't exist, or the path is wrong |
| `Failed to fetch` | The request was blocked, for example by a sandboxed preview. Try it on the real server |

**IPv6:** if your device reaches the server over IPv6, the error log shows a public address like `2603:...`. Add that address's first four groups as a `/64` to the `allow` list (for example `allow 2603:abcd:1234:5678::/64;`).

**Layout reverted or looks different on another device:** the page keeps a local copy in each browser and loads the server copy on startup. Make sure no "not synced" message is showing on the device where you made changes.
