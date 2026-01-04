# jake.town

Personal site and resume for Jake Swanson.

## Local Development

Just open `index.html` in a browser. No build step needed.

## Deployment

Hosted via GitHub Pages with custom domain.

### DNS Setup

Point your domain to GitHub Pages by adding these DNS records:

**For apex domain (jake.town):**
```
A     @    185.199.108.153
A     @    185.199.109.153
A     @    185.199.110.153
A     @    185.199.111.153
```

**Or for www subdomain:**
```
CNAME www  jakswa.github.io
```

### GitHub Pages Setup

1. Go to repo Settings > Pages
2. Source: Deploy from branch `main`
3. Custom domain: `jake.town`
4. Check "Enforce HTTPS"

## Exporting to PDF

Open the page in Chrome/Edge and print (Cmd+P / Ctrl+P). The print styles are optimized for a clean PDF export.

## Future Ideas

- [ ] Add a blog using a static site generator (Hugo, Eleventy, etc.)
- [ ] Add a `/projects` page with more detail on side projects
- [ ] Dark mode toggle
- [ ] Add a favicon
