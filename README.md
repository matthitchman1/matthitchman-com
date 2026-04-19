# matthitchman.com

Personal site and blog for Matt Hitchman. Honest writing about how AI actually works.

## What this is

A plain static site. No framework, no build step. Just HTML and CSS. Deploys to Azure Static Web Apps on every push to `main`.

## Structure

```
matthitchman-com/
├── index.html          # Home (hero, pillars, featured project, recent posts)
├── about.html          # About Matt
├── blog.html           # Blog index
├── 404.html            # Custom not-found page
├── posts/              # Blog posts (one HTML file each)
│   ├── welcome.html
│   ├── building-hiki.html
│   └── what-are-mcps.html
├── assets/
│   ├── styles.css      # All CSS — brand tokens as CSS custom properties
│   └── favicon.svg     # Brand favicon (MH mark)
├── staticwebapp.config.json   # Azure SWA routing / headers / 404 fallback
├── robots.txt
├── sitemap.xml
└── README.md           # This file
```

## How to write a new blog post

1. Duplicate any file in `posts/` and rename it. Lowercase, hyphens only (`my-new-post.html`).
2. Change the `<title>`, the meta description, the eyebrow, and the heading.
3. Write the post. Use `<h2>` for sections, `<h3>` for subsections, `<blockquote>` for pull quotes, `<code>` inline and `<pre><code>` for code blocks.
4. Add a card on `blog.html` (and optionally `index.html` under "Recent posts").
5. Add a `<url>` entry in `sitemap.xml`.
6. `git add . && git commit -m "post: my new post" && git push`. Azure deploys automatically.

## Brand

Brand guidelines V2 (April 2026) — visual identity (palette + typography) was originally designed for a separate project and is reused here. Tokens defined as CSS custom properties in `assets/styles.css`:

| Token | Hex | Use |
|---|---|---|
| `--dark-neutral` | `#1F2A2E` | Base background |
| `--primary-teal` | `#8FB9BC` | Secondary text, muted UI |
| `--bright-teal`  | `#76F0EB` | Accents, highlights, links |
| `--dark-teal`    | `#3E5C63` | Secondary accents |
| `--pop-orange`   | `#FF8A3D` | CTAs, emphasis |
| `--white`        | `#FFFFFF` | Primary text on dark |

Typeface: [Inter](https://fonts.google.com/specimen/Inter) (Google Fonts). Weights 300 / 400 / 600 / 700 / 800.

Voice: smart mate explaining something over coffee, not a professor behind a podium.

## Local preview

Just open `index.html` in a browser. Or, for relative paths to work the same as production:

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

## Deployment

Hosted on Azure Static Web Apps (free tier). GitHub Actions workflow — added automatically when the repo was connected to the SWA resource — deploys on every push to `main`.

## Domain & email

- **Domain**: registered at Cloudflare, DNS pointed at Azure SWA via CNAME (+ TXT for ownership verification). See DNS panel in Cloudflare.
- **Email**: `matt@matthitchman.com` forwards to personal Gmail via Cloudflare Email Routing (free). Gmail "Send mail as" configured for outbound replies.

## What's out of scope

- `hiki.matthitchman.com` — HIKI MCP tunnel (separate project, cloudflared-managed CNAME).
- `mcp.matthitchman.com` — HIKI OAuth Worker (Cloudflare Worker custom domain).

Don't touch those records.
