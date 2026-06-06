# Newtrick blog

A minimal personal blog using [bashblog](https://github.com/cfenollosa/bashblog) and [Caddy](https://caddyserver.com/), hosted on a Digital Ocean droplet.

No framework. No database. No build pipeline. Static HTML served directly.

## Philosophy

Inspired by the [1MB Club](https://1mb.club/) and [motherfuckingwebsite.com](https://motherfuckingwebsite.com/). The guiding principle is the **rule of least power** — use only what is needed to get the job done.

## Stack

- **[bashblog](https://github.com/cfenollosa/bashblog)** — a single bash script that converts Markdown to HTML and generates an index page, RSS feed, and tag pages automatically. Runs locally on macOS; only HTML is pushed to the server. I've made some changes to update it to HTML5 and inprove accessibility.
- **[Caddy](https://caddyserver.com/)** — a single binary web server that handles HTTPS automatically via Let's Encrypt. Config is minimal.
- **Digital Ocean droplet** — serves the static files.

## Local Setup

macOS ships with bash and Python 3, so the only dependency you will need installed is the markdown binary:

```zsh
brew install markdown
```

### Get bashblog

```sh
curl -o bb.sh https://raw.githubusercontent.com/cfenollosa/bashblog/master/bb.sh
chmod +x bb.sh
```

## Local Workflow

Write posts locally in Markdown, build with bashblog, preview with Python's built-in server, then rsync to the server.

### Preview locally

```sh
python3 -m http.server 8080
```

Site is available at `http://localhost:8080`.

### Write and publish a post

```sh
vim drafts/my-post-title.md
```

Write in standard Markdown. bashblog uses the first line as the post title if it is a `#` heading. Add tags on the last line:

```
Tags: writing, notes
```

Build the post:

```sh
./bb.sh post drafts/my-post-title.md
```

Then deploy:

```sh
rsync -av --exclude-from=".rsync-excludes" . user@yourdomain.com:/var/www/blog/
```

### Edit an existing post

```sh
vim drafts/my-post-title.md
./bb.sh edit my-post-title.html
rsync -av --exclude-from=".rsync-excludes" . user@yourdomain.com:/var/www/blog/
```

### Rebuild all

If the template or config changes:

```sh
./bb.sh rebuild all
rsync -av --exclude-from=".rsync-excludes" . user@yourdomain.com:/var/www/blog/
```

## Server Setup

### Install Caddy

```sh
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy
```

### Configure Caddy

Copy `Caddyfile` from the repo to `/etc/caddy/Caddyfile`, replacing `yourdomain.com` with your actual domain:

```
yourdomain.com {
    root * /var/www/blog
    encode zstd gzip
    try_files {path}.html {path}
    file_server

    header {
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
        Referrer-Policy strict-origin-when-cross-origin
    }
}

www.yourdomain.com {
    redir https://yourdomain.com{uri} permanent
}
```

- `try_files` lets `/about` serve `about.html` without the extension in the URL
- `encode` adds gzip/zstd compression automatically
- Caddy handles HTTPS via Let's Encrypt automatically
- www is redirected to the apex domain

Restart with `sudo systemctl restart caddy`.

### Set up the server directory

```sh
mkdir -p /var/www/blog
```

## Site Structure

```
./
  bb.sh              ← blog generator (run locally, never deployed)
  main.css           ← all styles
  footer.html        ← footer template (baked into HTML on rebuild)
  drafts/            ← markdown drafts (never deployed)
  index.html         ← generated post listing
  about.html         ← hand-written, deployed as-is
  feed.rss           ← generated RSS feed
  all_posts.html     ← generated archive
  all_tags.html      ← generated tag index
  tag_*.html         ← generated per-tag pages
  *.html             ← individual post pages
  Caddyfile          ← server config (deployed to /etc/caddy/, not /var/www/)
  .rsync-excludes    ← rsync filter list
```

## Styling

All styles live in `main.css`. The `footer.html` template is baked into every generated page at build time — the server only receives static HTML, no templates.

`bb.sh` is configured with `css_include=('main.css')` so only `main.css` is linked. No `blog.css`.

## What bashblog generates

- `index.html` — post listing (do not edit directly)
- `feed.rss` — RSS feed
- `tag_*.html` — one page per tag
- `blog.css`, `main.css` — default styles
- Individual post HTML files named from the post title and date

## Version Control

Keep `bb.sh`, `drafts/`, `css/`, `about.html`, and any header/footer template files in git. Generated HTML and RSS can be tracked too for a full history of the published site, or excluded if you only want to track source.
