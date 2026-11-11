# StudioLink

**Link Claude to Roblox Studio.** Once connected, Claude (on claude.ai, in a browser or the app) can read and edit the place you have open in Studio: browse the Explorer, change properties, build parts and models, write and edit scripts, read the Output window and run Luau.

It can also **sync your scripts to a GitHub repo**, so your game's code is backed up and versioned, and Claude can read it on GitHub too.

```
Claude ──custom connector──▶ Cloudflare Worker (relay) ◀──long-poll── StudioLink plugin (inside Studio)
```

Nothing extra runs on your computer. The relay lives on Cloudflare, and the plugin runs inside Studio.

---

## What you need

- Roblox Studio
- A free [Cloudflare](https://dash.cloudflare.com/sign-up) account, to host the relay
- A Claude plan that supports custom connectors

## Step 1: Deploy the relay (one time)

1. Download and unzip `studio-link.zip` from this repo.
2. In a terminal, go into the `worker` folder and deploy it:

   ```bash
   cd studio-link/worker
   npx wrangler deploy
   ```

   The first time, Wrangler opens a browser window so you can log in to Cloudflare.
3. Wrangler prints your relay URL, for example:

   ```
   https://studio-link.<your-subdomain>.workers.dev
   ```

   Open it in a browser. You should see **"Studio Link relay is running."**

The relay uses one Durable Object, which works on Cloudflare's free plan.

## Step 2: Install the plugin

1. Download **`StudioLink.rbxmx`** from the top of this repo. It's the newest plugin; don't use the copy inside the zip.
2. In Studio, open the **Plugins** tab and click **Plugins Folder**.
3. Drop `StudioLink.rbxmx` into that folder. Keep only **one** StudioLink file there: delete older copies such as `StudioLink-2.rbxmx`.
4. Quit Studio completely (⌘Q / Alt+F4) and open it again. A **Studio Link** button appears on the Plugins tab.

> Don't open or drag the `.rbxmx` into a place. That only adds an ordinary Script to your game; it doesn't install the plugin.

## Step 3: Connect Studio to the relay

1. Click **Studio Link** on the Plugins tab to open the panel.
2. In **box 1 (Relay URL)**, paste just your relay address: `https://studio-link.<your-subdomain>.workers.dev`
3. Press **Connect**.
4. Studio asks whether the plugin may make HTTP requests to your `workers.dev` domain. Click **Allow**.
5. Wait for the **green dot** ("Connected").

## Step 4: Add the connector in Claude

1. In the panel, click **Copy connector link**, then press **⌘C** (Mac) or **Ctrl+C** (Windows). The link looks like:

   ```
   https://studio-link.<your-subdomain>.workers.dev/mcp/<long-secret>
   ```

2. In Claude, go to **Settings → Connectors → Add custom connector**. Name it `Roblox Studio` and paste the link.
3. Start a **new chat**, turn the connector on (**+** → Connectors), and try:

   > Check studio_status, then show me what's in my Workspace.

From then on, the plugin reconnects automatically every time you open Studio.

---

## Step 5 (optional): Sync scripts to GitHub

StudioLink can save every script in your place to a GitHub repo as `.luau` files and load them back into Studio.

### 5a. Make a GitHub token

1. On GitHub, go to **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**.
2. Under **Repository access**, choose **Only select repositories** and pick the repo for your game.
   The repo needs at least one commit; a README is enough.
3. Under **Repository permissions**, set **Contents** to **Read and write**.
4. Generate the token and copy it.

### 5b. Give the token to your relay

In the `worker` folder:

```bash
npx wrangler secret put GITHUB_TOKEN
# paste the token when asked
npx wrangler deploy
```

The token is stored as an encrypted secret on Cloudflare. It is never put in the plugin or sent to Studio.

### 5c. Pick the repo in Studio

In the Studio Link panel, under **3. GitHub sync**:

1. Enter the repo as `owner/name`, for example `blockblock2/MyGame`, and the branch (`main` by default).
2. Click **Push to GitHub**. The panel shows how many files were added, changed or removed.

| Button | What it does |
|---|---|
| **Push to GitHub** | Commits every script in the place. Only changed files go into the commit, and scripts you deleted in Studio are removed from the repo. |
| **Pull from GitHub** | Loads the repo's scripts into Studio, updating existing scripts and creating missing ones (missing folders are created too). It **never deletes** anything in Studio, and you can undo it with ⌘Z / Ctrl+Z. |
| **Auto-push** | When ON, pushes every 5 minutes while connected. It only makes a commit when something actually changed. |

Claude can do the same with the `github_push` and `github_pull` tools, for example: *"push my scripts to GitHub with the message 'add round system'"*.

### What the repo looks like

Scripts are saved in a `place/` folder that mirrors the Explorer:

```
place/
├── ServerScriptService/
│   └── RoundManager.server.luau     ← Script
├── ReplicatedStorage/
│   └── Modules/
│       └── Util.luau                ← ModuleScript
├── StarterPlayer/
│   └── StarterPlayerScripts/
│       └── Input.client.luau        ← LocalScript
└── studio-link.json                 ← maps each file back to its exact place in Studio
```

Only scripts are synced. Parts, models, GUIs and other objects stay in your place file, and anything outside `place/` in the repo is left untouched.

---

## What Claude can do

| Tool | What it does |
|---|---|
| `studio_status` | Checks whether Studio is connected and which place is open |
| `get_tree` | Shows the Explorer hierarchy |
| `get_properties` / `set_properties` | Reads or changes properties, attributes and tags |
| `create_instance` / `clone_instance` / `delete_instance` | Builds or removes parts, models, GUIs and scripts |
| `read_script` / `write_script` / `edit_script` | Works with scripts |
| `search` | Finds things by name, class, or text inside scripts |
| `get_selection` / `set_selection` | Reads what you've selected, or selects things to show you |
| `get_output` | Reads the Output window (prints, warnings, errors) |
| `run_luau` | Runs any Luau code in edit mode |
| `github_push` / `github_pull` | Syncs scripts with your GitHub repo |

Every change appears in Studio's undo history as **"Claude: …"**, so **⌘Z / Ctrl+Z** undoes it.

## Keep your link private

The connector link contains a long random secret. **Anyone who has it can edit whatever place you have open** while Studio is connected. Don't post it or share it. If it leaks, click **Make a new secret URL** in the panel and update the connector in Claude.

Reinstalling the plugin also creates a new secret, so update the connector in Claude afterwards.

## Troubleshooting

| What you see | What to do |
|---|---|
| **"Roblox HTTP limit hit"** | Another copy of the plugin is probably running. Quit Studio, leave only one `StudioLink.rbxmx` in the Plugins folder, reopen, and press Connect. |
| **"HTTP blocked"** | Go to Plugins → **Manage Plugins** → Studio Link and allow HTTP access to your `workers.dev` domain. |
| **"That URL isn't a Studio Link relay"** | Box 1 should be your relay address only, not the connector link or another site. |
| **The Studio Link button is missing** | Stop any Play test (the plugin hides during Play), then check the Plugins folder and look in **View → Output** for red errors mentioning StudioLink. |
| **Claude says Studio isn't connected** | Make sure the panel shows the green dot and that Claude's connector uses the *current* link from box 2. |
| **Claude says "MCP server not allowed"** | The connector was changed or removed during the chat. Start a new chat. |
| **GitHub: "relay has no GitHub token"** | Run `npx wrangler secret put GITHUB_TOKEN`, then `npx wrangler deploy` (step 5b). |
| **GitHub: 401** | The token is wrong or expired. Make a new one and put it again. |
| **GitHub: 403 / 404** | Check the repo name, and that the token can access that repo with Contents read and write. |
| **GitHub: 409 "repo is empty"** | Add a README to the repo on GitHub first. |
| **A tool timed out** | The command may still have run. Ask Claude to check before it retries. |

## How it works

- The **plugin** keeps one request open to the relay (about 20 seconds at a time) and waits for work. When Claude calls a tool, the relay hands the command to the plugin. The plugin runs it in Studio and posts the result back.
- The **relay** (`worker/src/index.js`) is a Cloudflare Worker that speaks MCP over Streamable HTTP. A Durable Object, keyed by your secret, connects Claude's calls to your Studio session.
- **GitHub sync** uses GitHub's Git Data API to make one commit per push with only the changed files. Pull reads all files in one GraphQL request.
- The relay stores nothing. Commands and results pass through memory only, and your GitHub token is kept as a Cloudflare secret.
