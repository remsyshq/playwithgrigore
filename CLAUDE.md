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

Config file: `/etc/nginx/sites-enabled/playwithgrigore.com`

Clean URLs are handled via nginx `alias` directives (no symlinks needed):

```nginx
# Game aliases - clean URLs
location /2048/ {
    alias /var/www/playwithgrigore.com/games/2048/;
}
location /tetris/ {
    alias /var/www/playwithgrigore.com/games/tetris/;
}
location /snake/ {
    alias /var/www/playwithgrigore.com/games/snake/src/;
}
location /pacman/ {
    alias /var/www/playwithgrigore.com/games/pacman/;
}
location /clumsy-bird/ {
    alias /var/www/playwithgrigore.com/games/clumsy-bird/;
}
location /hexgl/ {
    alias /var/www/playwithgrigore.com/games/hexgl/;
}
```

This means `https://playwithgrigore.com/tetris/` serves `games/tetris/index.html`.

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

### Adding a New Game

1. Clone/create game in `games/<game-name>/`
2. Ensure it has an `index.html` entry point
3. Add mobile controls if needed (see Mobile section)
4. Update `index.html` landing page with link to new game
5. Commit, push, and deploy
6. On server, add nginx alias for clean URL:
   ```nginx
   location /<game-name>/ {
       alias /var/www/playwithgrigore.com/games/<game-name>/;
       try_files $uri $uri/ =404;
   }
   ```
7. Test and reload nginx: `nginx -t && systemctl reload nginx`

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
- Clean URLs like `/tetris/` are handled by nginx aliases (no symlinks)
- Snake game entry point is `games/snake/src/` (served at `/snake/`)
- Always test on mobile after making changes
