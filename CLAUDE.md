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

## Adding/Editing Games

### Folder Structure Rules

All games MUST be stored in the `games/` directory:

```
games/
├── 2048/           # Puzzle game (has mobile controls)
├── tetris/         # Classic tetris (has mobile controls)
├── snake/          # Snake game (entry: src/index.html)
├── pacman/         # Pac-man clone
├── clumsy-bird/    # Flappy bird clone
└── hexgl/          # 3D racing game (Three.js)
```

### Server Symlinks

On the server, symlinks in root point to `games/` for clean URLs:
- `/2048` → `games/2048`
- `/tetris` → `games/tetris`
- etc.

This means URLs like `https://playwithgrigore.com/tetris/` work.

### Adding a New Game

1. Clone/create game in `games/<game-name>/`
2. Ensure it has an `index.html` entry point
3. Add mobile controls if needed (see Mobile section)
4. Update `index.html` landing page with link to new game
5. Commit, push, and deploy
6. On server, create symlink: `ln -s games/<game-name> <game-name>`

### Mobile Controls

Games should work on mobile. We've added touch controls to:
- **Tetris**: Touch buttons below canvas (LEFT, ROTATE, DROP, RIGHT)
- **2048**: Arrow buttons below grid (swipe also works)

Pattern for mobile controls:
- Detect mobile: `@media (pointer: coarse)` or `max-width: 768px`
- Use `touch-action: manipulation` on buttons
- Use `touchstart` with `{ passive: false }` and `e.preventDefault()`

## Notes

- The `.env` file is gitignored and should not be committed
- Games are linked from `/games/<game-name>/` paths
- Snake game entry point is at `/games/snake/src/`
- Always test on mobile after making changes
