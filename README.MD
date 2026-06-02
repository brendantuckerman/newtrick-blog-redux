# Personal Blog

A minimal personal blog using [bashblog](https://github.com/cfenollosa/bashblog) and [Caddy](https://caddyserver.com/), hosted on a Digital Ocean droplet.

No framework. No database. No build pipeline. Static HTML served directly.

## Philosophy

Inspired by the [1MB Club](https://1mb.club/) and [motherfuckingwebsite.com](https://motherfuckingwebsite.com/). The guiding principle is the **rule of least power** — use only what is needed to get the job done. Sub-1MB page weight is a useful heuristic, not a hard constraint.

If a tool is being considered, the first question is whether it is needed at all.

## Stack

- **[bashblog](https://github.com/cfenollosa/bashblog)** — a single bash script that converts Markdown to HTML and generates an index page, RSS feed, and tag pages automatically. Runs locally; only HTML is pushed to the server.
- **[Caddy](https://caddyserver.com/)** — a single binary web server that handles HTTPS automatically via Let's Encrypt. Config is minimal.
- **Digital Ocean droplet** — serves the static files.

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

### Set Up the Server Directory

```sh
mkdir -p /var/www/blog
```

No other server-side tooling is needed. The server only serves static files.

## Local Setup

### Install bashblog

```sh
mkdir -p ~/blog
cd ~/blog
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

Posts are written locally in Markdown, processed by bashblog locally, and only the generated HTML is pushed to the server.

### Write and Publish a Post

```sh
vim ~/blog/drafts/my-post-title.md
```

Write in standard Markdown. bashblog uses the first line as the post title if it is a `#` heading, otherwise it prompts. Add tags on the last line:

```
Tags: writing, notes
```

Then process and deploy:

```sh
cd ~/blog
./bb.sh post drafts/my-post-title.md
rsync -av --exclude="*.md" --exclude="bb.sh" . user@yourdomain.com:/var/www/blog/
```

This can be wrapped in a shell function:

```sh
publish() {
  local file="$1"
  cd ~/blog
  ./bb.sh post "$file"
  rsync -av --exclude="*.md" --exclude="bb.sh" . user@yourdomain.com:/var/www/blog/
}
```

Add to `~/.zshrc`, then: `publish drafts/my-post-title.md`

### Edit an Existing Post

```sh
vim ~/blog/drafts/my-post-title.md
cd ~/blog && ./bb.sh edit my-post-title.html
rsync -av --exclude="*.md" --exclude="bb.sh" . user@yourdomain.com:/var/www/blog/
```

### Rebuild All

If the template or config changes:

```sh
cd ~/blog && ./bb.sh rebuild all
rsync -av --exclude="*.md" --exclude="bb.sh" . user@yourdomain.com:/var/www/blog/
```

## Site Structure

```
~/blog/
  bb.sh
  drafts/
    my-post-title.md
  index.html       ← bashblog-generated post listing, serves as /
  about.html       ← hand-written, served at /about
  feed.rss         ← bashblog-generated
  tag_*.html       ← bashblog-generated
  *.html           ← individual posts
```

`about.html` is a plain HTML file — no template, no tooling. Write it once and update as needed. Link to it from bashblog's template (see `global_header` in `bb.sh`).

The server mirrors this structure at `/var/www/blog/`, minus the drafts and source files.

## What bashblog Generates

- `index.html` — post listing
- `feed.rss` — RSS feed
- `tag_*.html` — one page per tag
- Individual post HTML files named from the post title and date

## Keeping Drafts in Version Control (Optional)

Keep `~/blog/` as a git repo. Add a `.gitignore` to exclude generated HTML if you only want to track source:

```
*.html
feed.rss
tag_*.html
```

Or track everything for a full history of the published site.
