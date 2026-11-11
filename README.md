# StudioLink

**Link Claude to Roblox Studio.** Once connected, Claude (on claude.ai, in a browser or the app) can read and edit the place you have open in Studio: browse the Explorer, change properties, build parts and models, write and edit scripts, read the Output window and run Luau.

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
| **A tool timed out** | The command may still have run. Ask Claude to check before it retries. |

## How it works

- The **plugin** keeps one request open to the relay (about 20 seconds at a time) and waits for work. When Claude calls a tool, the relay hands the command to the plugin. The plugin runs it in Studio and posts the result back.
- The **relay** (`worker/src/index.js`) is a Cloudflare Worker that speaks MCP over Streamable HTTP. A Durable Object, keyed by your secret, connects Claude's calls to your Studio session.
- No data is stored. Commands and results pass through memory only.
