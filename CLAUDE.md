# playwithgrigore.com

## Repository

GitHub: https://github.com/remsyshq/playwithgrigore

## Local Development

The project is a static website with HTML games. No build process required.

### Directory Structure

```
/
├── index.html          # Main landing page
├── games/              # Game subdirectories
│   ├── 2048/
│   ├── tetris/
│   ├── snake/src/      # Snake game entry point
│   ├── pacman/
│   ├── clumsy-bird/
│   └── hexgl/
├── .gitignore
└── CLAUDE.md
```

## Deployment

### Server Details

- **Server**: wp.retently.com
- **SSH**: `ssh root@wp.retently.com`
- **Web Root**: `/var/www/playwithgrigore.com`
- **Domain**: https://playwithgrigore.com

### Nginx Configuration

The site is served by nginx with the following config (located at `/etc/nginx/sites-enabled/`):

```nginx
server {
    server_name playwithgrigore.com www.playwithgrigore.com;
    root /var/www/playwithgrigore.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # SSL managed by Certbot
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/playwithgrigore.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/playwithgrigore.com/privkey.pem;
}
```

### Deploy Process

1. **Push changes to GitHub**:
   ```bash
   git add -A
   git commit -m "Your commit message"
   git push origin master
   ```

2. **SSH to server and run deploy script**:
   ```bash
   ssh root@wp.retently.com
   /var/www/playwithgrigore.com/deploy.sh
   ```

   Or in one command:
   ```bash
   ssh root@wp.retently.com "/var/www/playwithgrigore.com/deploy.sh"
   ```

### Deploy Script

Located at `/var/www/playwithgrigore.com/deploy.sh`:

```bash
#!/bin/bash
cd /var/www/playwithgrigore.com
git fetch origin
git reset --hard origin/master
echo "Deployed at $(date)"
```

## Notes

- The `.env` file is gitignored and should not be committed
- Games are linked from `/games/<game-name>/` paths
- Snake game entry point is at `/games/snake/src/`
