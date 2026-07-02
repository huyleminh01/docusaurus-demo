# Deployment (GitHub Pages + custom domain)

This site is served on a custom domain (`blog.dbdiagram.*`) via GitHub Pages
using the GitHub Actions artifact flow (`upload-pages-artifact` +
`deploy-pages`). See [.github/workflows/deploy.yml](.github/workflows/deploy.yml).

## Key fact

`baseUrl` in [docusaurus.config.ts](docusaurus.config.ts) is **only a URL
prefix** written into the generated HTML (links, `<script src>`, assets). It does
**not** nest the build output — `build/` is always flat (`build/assets/...`).

With a **custom domain**, GitHub Pages serves the uploaded artifact at the
**domain root** (`/`) — there is no path prefix. So `baseUrl` must match wherever
the files physically end up in the artifact, or every asset 404s.

`baseUrl` is env-driven: `process.env.BLOG_BASE_URL || '/'` (default `/` for local
dev; CI overrides it).

## Approaches

### 1. Root — `baseUrl: '/'`

Build normally and upload `build/`. Site serves at `https://<domain>/`.
Simplest; use this if you don't need a `/blog/` path.

```yaml
- run: npm run build            # baseUrl defaults to '/'
- uses: actions/upload-pages-artifact@v3
  with: { path: build }
```

### 2. Dual build — serve both `/` and `/blog/`

Build twice and merge the `/blog/` build under `build/blog/`. Serves the root
site **and** a `/blog/` copy from one artifact. (See the commented block in the
workflow.)

```yaml
- run: |
    npm run build                                        # root -> build/
    BLOG_BASE_URL=/blog/ npm run build -- --out-dir tmp  # /blog/ build
    mkdir -p build/blog && cp -R tmp/. build/blog/ && rm -rf tmp
- uses: actions/upload-pages-artifact@v3
  with: { path: build }
```

### 3. Path only — `baseUrl: '/blog/'`, nest the artifact **(current)**

Build for `/blog/`, then nest the flat `build/` under a `blog/` folder so the
custom-domain root serves everything at `/blog/`. A tiny root `index.html`
redirects the bare domain to `/blog/`.

```yaml
- run: |
    BLOG_BASE_URL=/blog/ npm run build
    mkdir -p _site/blog
    cp -r build/. _site/blog/     # note the `/.` — copies .nojekyll too
    echo '<!doctype html><meta http-equiv="refresh" content="0; url=/blog/">' > _site/index.html
- uses: actions/upload-pages-artifact@v3
  with: { path: _site }
```

Result: `https://<domain>/blog/` works; `https://<domain>/` redirects to it.

## Custom domain note

The custom domain must be set in **repo Settings → Pages** (persisted
server-side by the Actions deployment). The `url` field in the config should
match the actual domain you serve from.
