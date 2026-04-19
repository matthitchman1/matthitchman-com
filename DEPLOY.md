# Deploy matthitchman.com — step-by-step

**Goal:** ship the site at `https://matthitchman.com` and turn on `matt@matthitchman.com` email forwarding. Expected time: 30–45 minutes end-to-end, most of which is waiting for DNS propagation.

**Phases**
1. Put the code on GitHub
2. Create an Azure Static Web App and connect it to the GitHub repo
3. Point `matthitchman.com` + `www.matthitchman.com` at Azure via Cloudflare DNS
4. Turn on Cloudflare Email Routing for `matt@matthitchman.com`
5. Set up Gmail "Send mail as" so you can reply from the custom address
6. Verify the whole thing works end-to-end

**Jargon one-liners (for reference)**

- **Repo** — a folder of code tracked by git, hosted on GitHub.
- **CNAME record** — a DNS entry that says "this subdomain is an alias of that hostname." (e.g. `www.matthitchman.com` → `xxx.azurestaticapps.net`).
- **A record** — a DNS entry that maps a name to a specific IP address.
- **TXT record** — a DNS entry that holds arbitrary text, usually used to prove you own the domain.
- **Apex / root domain** — `matthitchman.com` with no subdomain in front. The opposite of `www.matthitchman.com`.
- **MX record** — a DNS entry that says "mail for this domain goes to that server."
- **TTL** — "time to live." How long DNS servers around the world should cache a record before re-checking. Leave at default (Auto / 300s).

---

## Phase 1 — Put the code on GitHub

**Goal:** create a new repo called `matthitchman-com` under your GitHub account, and push this entire folder to it.

### 1.1 — Create the empty repo on github.com

1. Open **[https://github.com/new](https://github.com/new)**.
2. Fill in the form:
   - **Repository name:** `matthitchman-com`
   - **Description:** `Personal site — matthitchman.com`
   - **Public** or **Private** — either is fine (Azure Static Web Apps supports both on the free tier). Public is simpler for a personal site.
   - **Do NOT tick** "Add a README", "Add .gitignore", or "Choose a license." We already have those.
3. Click **Create repository**.
4. You'll land on a page with setup instructions. Leave this tab open — we'll use a URL from it in the next step.

### 1.2 — Push your files to the repo

You have two options. **Pick one.**

#### Option A (recommended) — Terminal / git

Open Terminal on your Mac and run each block one at a time. Replace `YOUR_USERNAME` in the `remote add` line with your actual GitHub username (if you haven't told me already, type it in chat and I'll update this doc).

```bash
# Navigate into the folder
cd ~/Documents/matthitchman-com   # <-- adjust if your folder lives elsewhere

# Initialise git and make the first commit
git init
git add .
git commit -m "initial site"
git branch -M main

# Connect the local repo to GitHub
git remote add origin https://github.com/YOUR_USERNAME/matthitchman-com.git

# Push
git push -u origin main
```

**First-time auth.** On the `git push` step, macOS may pop up a login window or ask for a password. Don't use your GitHub password — GitHub disabled that years ago. Two cleanest ways to authenticate:

- **Easiest**: install GitHub CLI once, and it handles auth for you. In Terminal:
  ```bash
  brew install gh       # needs Homebrew — brew.sh if you don't have it
  gh auth login         # pick GitHub.com → HTTPS → Yes → Login with browser
  ```
  Then re-run `git push -u origin main`.
- **Manual**: create a "Personal Access Token (classic)" at [github.com/settings/tokens](https://github.com/settings/tokens) with the `repo` scope, and paste it as the password when prompted.

**Expected result:** Terminal reports `Branch 'main' set up to track remote branch 'main' from 'origin'.` Refresh the GitHub repo page in your browser — all the files should be there.

#### Option B (zero-terminal fallback) — Drag-and-drop in browser

Only use this if the terminal path is giving you grief.

1. On the empty-repo page you saw after 1.1, click the link **"uploading an existing file"** (it's in the middle of the setup instructions).
2. Finder → open your `matthitchman-com` folder → select **all contents** (Cmd+A) → drag them into the GitHub browser window.
3. Commit message: `initial site` → click **Commit changes**.

**Caveat:** this path uploads the files but doesn't set up a local git repo on your Mac. For future edits you'd either edit in the GitHub web UI or set up git locally later. For the very first push, it works fine.

---

## Phase 2 — Azure Static Web App

**Goal:** create an Azure Static Web App (SWA) resource that hosts the site and redeploys automatically every time you push to GitHub. Free tier.

### 2.1 — Create the SWA resource

1. Open **[https://portal.azure.com](https://portal.azure.com)** and sign in with your Microsoft work identity.
2. In the top search bar, type `Static Web Apps` → click **Static Web Apps** (the blue-and-white logo).
3. Click **+ Create** (top-left).
4. Fill in the **Basics** tab:

   | Field | Value |
   |---|---|
   | **Subscription** | Pick any subscription available to your account (Azure will list them). If there's more than one, use a personal-flavoured one rather than a customer subscription if you have the option. |
   | **Resource group** | Click **Create new** → name it `matthitchman-com-rg` → OK. (A resource group is just a folder for related Azure resources.) |
   | **Name** | `matthitchman-com` |
   | **Plan type** | **Free** |
   | **Region for Azure Functions API** | **East Asia** (closest to NZ — doesn't really matter for a static site without a backend API) |
   | **Source** | **GitHub** |

5. Under **Deployment details**, click **Sign in with GitHub** and authorize Azure to read your repos. Then pick:

   | Field | Value |
   |---|---|
   | **Organization** | Your GitHub username |
   | **Repository** | `matthitchman-com` |
   | **Branch** | `main` |

6. Under **Build Details**:

   | Field | Value |
   |---|---|
   | **Build Presets** | **Custom** |
   | **App location** | `/` |
   | **Api location** | *(leave blank)* |
   | **Output location** | *(leave blank)* |

7. Click **Review + create** → then **Create**.

8. Azure spins for 60–90 seconds, then says "Your deployment is complete." Click **Go to resource**.

### 2.2 — Confirm it deployed

1. On the resource Overview page, find the **URL** field — it looks like `https://polite-mushroom-01234abcd.xxxxx.azurestaticapps.net`. Click it.
2. You should see the site rendering with the orange "actually" highlight and the dark teal background.
3. If you get a "Waiting for content" placeholder, the GitHub Actions workflow is still running. Wait 2–3 minutes and refresh. (You can also check the **Actions** tab in your GitHub repo — there should be a green checkmark when the build finishes.)

**Copy that `xxxxx.azurestaticapps.net` hostname somewhere** — you'll need it in Phase 3.

---

## Phase 3 — Point matthitchman.com at Azure

**Goal:** configure DNS in Cloudflare so `matthitchman.com` and `www.matthitchman.com` both serve the Azure site, with SSL auto-issued by Azure.

⚠️ **Do not touch** the existing `hiki` and `mcp` CNAME records. Those belong to your HIKI project and must stay as they are.

### 3.1 — Add the `www` subdomain first (easier)

1. In Azure portal, still on your SWA resource, in the left nav click **Custom domains** → **+ Add**.
2. Choose **Custom domain on other DNS** → **Add**.
3. **Domain name:** `www.matthitchman.com` → Next.
4. On the **Validate + add** screen:
   - **Hostname record type:** `CNAME`
   - Azure will show you a value to point the CNAME at (it'll look like `polite-mushroom-xxxxxx.1.azurestaticapps.net`).
5. **Don't click Add yet.** Leave this tab open and switch to Cloudflare.

**Now in Cloudflare:**

1. Open **[https://dash.cloudflare.com](https://dash.cloudflare.com)** → click the `matthitchman.com` zone → left nav → **DNS** → **Records**.
2. Click **+ Add record**:

   | Field | Value |
   |---|---|
   | **Type** | `CNAME` |
   | **Name** | `www` |
   | **Target** | the `...azurestaticapps.net` hostname from Azure |
   | **Proxy status** | **DNS only** (grey cloud, not orange) — important; the orange proxy breaks Azure's SSL validation |
   | **TTL** | Auto |

3. Click **Save**.

**Back in Azure:** click **Add** on the custom-domain dialog. Azure will validate the CNAME (takes 30–120 seconds), then start issuing an SSL cert.

### 3.2 — Add the apex (`matthitchman.com`)

Apex domains are trickier because you can't normally have a CNAME on the root. Cloudflare has a feature called **CNAME flattening** that makes it work anyway, which is why we're doing this here and not at a different registrar.

1. Back in Azure SWA **Custom domains**, click **+ Add** again.
2. Choose **Custom domain on other DNS** → **Add**.
3. **Domain name:** `matthitchman.com` (no `www`) → Next.
4. On the **Validate + add** screen:
   - **Hostname record type:** `TXT` (Azure will ask you to prove ownership of the apex before it issues the cert).
   - Azure shows you a **TXT value** (a long random string starting with something like `_dnsauth.matthitchman.com`).
   - Copy the TXT **Name** and **Value** Azure shows.
5. **Don't click Add yet.** Switch to Cloudflare.

**In Cloudflare DNS:**

1. Click **+ Add record**:

   | Field | Value |
   |---|---|
   | **Type** | `TXT` |
   | **Name** | paste the name from Azure (if Azure gives `_dnsauth`, Cloudflare will auto-complete it as `_dnsauth.matthitchman.com`) |
   | **Content** | paste the value from Azure |
   | **TTL** | Auto |

2. Save.

**Back in Azure:** click **Add** — Azure validates the TXT. Once it's validated, Azure will now ask for a **CNAME** on the apex pointing at the same `...azurestaticapps.net` hostname as `www`.

**In Cloudflare DNS:**

1. Click **+ Add record**:

   | Field | Value |
   |---|---|
   | **Type** | `CNAME` |
   | **Name** | `@` (this means the apex — `matthitchman.com` itself) |
   | **Target** | the same `...azurestaticapps.net` hostname |
   | **Proxy status** | **DNS only** (grey cloud) |
   | **TTL** | Auto |

2. Save. Cloudflare's CNAME flattening handles the fact that a CNAME on the apex is technically unusual.

### 3.3 — Wait for SSL, then verify

Azure needs 5–15 minutes to issue the SSL certificate for both hostnames. In the Azure portal **Custom domains** list, each entry will go through: `Validating → Adding → Ready`.

Once both show **Ready**:

1. Open `https://matthitchman.com` → site loads, padlock is green.
2. Open `https://www.matthitchman.com` → site loads, padlock is green.

**Optional but nice:** add a redirect so `www` always redirects to the apex (or vice versa) for a single canonical URL. This is done in Cloudflare under **Rules → Redirect Rules**. Skip for now if you want; it's polish.

---

## Phase 4 — Email forwarding (Cloudflare Email Routing)

**Goal:** email sent to `matt@matthitchman.com` forwards to `matthitchman1@gmail.com`. Free. Unlimited addresses.

### 4.1 — Enable Email Routing

1. In Cloudflare dashboard → `matthitchman.com` zone → left nav → **Email** → **Email Routing**.
2. Click **Get started** (if first time) → Cloudflare proposes a set of MX + TXT records to add. Click **Add records and enable**.
3. ⚠️ **If Cloudflare warns that existing MX records conflict:** you likely have no other MX records right now (because you haven't set up email before), so this shouldn't trigger. If it does, screenshot the warning and send it to me before accepting.

### 4.2 — Add the forwarding rule

1. In Email Routing, go to **Routing rules** tab → **Custom addresses** → **Create address**.
2. Fill in:
   - **Custom address:** `matt`
   - **Action:** **Send to an email**
   - **Destination:** `matthitchman1@gmail.com`
3. Click **Save**.
4. Cloudflare sends a verification email to `matthitchman1@gmail.com`. Open it → click **Verify email address**.
5. Back in Cloudflare, the rule status switches to **Active**.

### 4.3 — Send yourself a test

1. From any other email account, send a message to `matt@matthitchman.com`.
2. It should arrive in your Gmail inbox within ~30 seconds.

If it bounces or doesn't arrive:

- Re-check the MX records in Cloudflare DNS → **Email Routing** section should show three MX records pointing at `route1/2/3.mx.cloudflare.net`.
- Double-check the destination address was verified (not just entered).

---

## Phase 5 — Gmail "Send mail as"

**Goal:** when you reply to an email at `matt@matthitchman.com`, the reply goes out with that address as the From, not `matthitchman1@gmail.com`.

1. Open Gmail → click the **gear icon** (top right) → **See all settings**.
2. Click the **Accounts and Import** tab.
3. In the **Send mail as** row, click **Add another email address**.
4. In the popup:
   - **Name:** `Matt Hitchman`
   - **Email address:** `matt@matthitchman.com`
   - ✅ Keep **"Treat as an alias"** ticked.
   - **Next Step** → On the SMTP screen, leave everything as default (Gmail's own SMTP) → **Next Step**.
5. Gmail asks you to verify by sending a confirmation code to `matt@matthitchman.com`. Because of Phase 4, that email will arrive at your Gmail inbox within seconds. Click the verification link in it (or paste the code in the Gmail popup).
6. Back in **Accounts and Import** → **Send mail as** row, click **make default** next to `matt@matthitchman.com` if you want all new outgoing mail to come from there.

7. Test: compose a new email → the **From** field now has a dropdown letting you choose `matt@matthitchman.com`. Send a test to a friend or your work email. Check that the sender shows correctly.

---

## Phase 6 — Final verification checklist

Work through these. If any fail, ping me and I'll debug.

- [ ] `https://matthitchman.com` loads the site, padlock green, no mixed-content warnings.
- [ ] `https://www.matthitchman.com` loads the site, padlock green.
- [ ] All three blog post links work: `/posts/welcome.html`, `/posts/building-hiki.html`, `/posts/what-are-mcps.html`.
- [ ] `/nope` (or any bad path) serves the custom 404 page with the "Nothing here." heading.
- [ ] Sending an email to `matt@matthitchman.com` arrives in Gmail within 30 seconds.
- [ ] Composing a new Gmail and choosing `matt@matthitchman.com` in the From dropdown, sending to another account, shows the custom address as the sender.
- [ ] Mobile view — open `https://matthitchman.com` on your phone. Hero, pillars, posts all stack cleanly.

**Untouched (verify still fine):**

- [ ] `https://hiki.matthitchman.com` — HIKI MCP tunnel still reachable (test from Claude Desktop or however you normally hit it).
- [ ] `https://mcp.matthitchman.com` — HIKI OAuth gateway still responds.

---

## What to do after it's live

- Write a new post: duplicate any file in `posts/`, edit, add a card to `blog.html` + `index.html` + an entry in `sitemap.xml`, then `git add . && git commit -m "post: thing" && git push`. Azure redeploys in ~90 seconds.
- Add your photo to the About page if you want — I left that out because I don't have one. Drop a JPG into `assets/` and I'll wire it in.
- Upgrade to Google Workspace later if you want a real mailbox at `matt@matthitchman.com` instead of forwarding — that's a separate flow via Google (around $7/mo), and it means replacing the Cloudflare MX records with Google's.

---

## Troubleshooting quickies

**"My push is hanging on `git push` and nothing's happening."**
You're probably stuck on auth. Ctrl-C out, then run `gh auth login` (if you have GitHub CLI) or create a Personal Access Token.

**"Azure Static Web App build is failing in GitHub Actions."**
Check the Actions tab on your GitHub repo → click the failed run → look for the red step. Most likely issue: the `output_location` or `app_location` doesn't match what we set in 2.1. For this site both should be `/` and `blank` respectively, because there's no build step.

**"SSL is taking forever on `matthitchman.com`."**
Azure needs the TXT validation record to be reachable AND the CNAME target to resolve before it'll issue the cert. Run `dig matthitchman.com TXT +short` — if it returns nothing, the Cloudflare record isn't live yet (wait a few more minutes).

**"The site is showing the old content after I've pushed a change."**
Two possible causes: (1) GitHub Actions hasn't finished — check the repo's Actions tab. (2) Browser cache — hit Cmd+Shift+R to hard-refresh.

**"I don't want the Technical Hitch-style brand circles / orange highlight anymore."**
All the visual tokens live at the top of `assets/styles.css` — search for `:root { ... --pop-orange` and change colours there. Everything else follows.
