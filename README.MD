# Personal Blog

A minimal personal blog using [bashblog](https://github.com/cfenollosa/bashblog) and [Caddy](https://caddyserver.com/), hosted on a Digital Ocean droplet.

No framework. No database. No build pipeline. Static HTML served directly.

## Philosophy

Inspired by the [1MB Club](https://1mb.club/) and [motherfuckingwebsite.com](https://motherfuckingwebsite.com/). The guiding principle is the **rule of least power** — use only what is needed to get the job done. Sub-1MB page weight is a useful heuristic, not a hard constraint.

If a tool is being considered, the first question is whether it is needed at all.

## Stack

- **[bashblog](https://github.com/cfenollosa/bashblog)** — a single bash script that converts Markdown to HTML and generates an index page, RSS feed, and tag pages automatically. Runs locally on macOS; only HTML is pushed to the server.
- **[Caddy](https://caddyserver.com/)** — a single binary web server that handles HTTPS automatically via Let's Encrypt. Config is minimal.
- **Digital Ocean droplet** — serves the static files.

## Local Setup

No dependencies to install. macOS ships with bash and Python 3.

### Get bashblog

```sh
curl -o bb.sh https://raw.githubusercontent.com/cfenollosa/bashblog/master/bb.sh
chmod +x bb.sh
```

Edit the config variables at the top of `bb.sh`:

```sh
global_title="My Blog"
global_description="Notes and writing"
global_author="Your Name"
global_url="https://yourdomain.com"
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

`/etc/caddy/Caddyfile`:

```
yourdomain.com {
    root * /var/www/blog
    try_files {path}.html {path}
    file_server
}
```

`try_files` lets `/about` serve `about.html` without requiring the extension in the URL.

Caddy handles SSL automatically. Restart with `sudo systemctl restart caddy`.

### Set up the server directory

```sh
mkdir -p /var/www/blog
```

## Site Structure

```
~/blog/
  bb.sh
  css/
    style.css        ← hand-written styles
  drafts/
    my-post-title.md
  index.html         ← bashblog-generated post listing
  about.html         ← hand-written, served at /about
  feed.rss           ← bashblog-generated
  tag_*.html         ← bashblog-generated
  *.html             ← individual posts
```

`about.html` is a plain HTML file — no template, no tooling. Write it once and update as needed.

## Styling

bashblog wraps all generated pages with a header and footer. Set `header_file` and `footer_file` in `bb.sh` to point to your own HTML snippet files, then run `./bb.sh rebuild all`. This lets you link your own stylesheet and control the page structure.

bashblog also generates `blog.css` and `main.css` with default styles. Replace or ignore these in favour of `css/style.css`.

## What bashblog generates

- `index.html` — post listing (do not edit directly)
- `feed.rss` — RSS feed
- `tag_*.html` — one page per tag
- `blog.css`, `main.css` — default styles
- Individual post HTML files named from the post title and date

## Version Control

Keep `bb.sh`, `drafts/`, `css/`, `about.html`, and any header/footer template files in git. Generated HTML and RSS can be tracked too for a full history of the published site, or excluded if you only want to track source.
