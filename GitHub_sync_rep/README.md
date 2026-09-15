# 🔄 n8n → GitHub Auto-Sync ())(( fienfie))efefef

**Stop exporting workflow JSON by hand.** One click, and your n8n workflow, README, and screenshots land in GitHub — with zero duplicate commits and zero manual file wrangling.

Built as a self-hosted alternative to n8n's Enterprise-only "Source Control" feature, using nothing but n8n itself.

---

## Why this exists

n8n's built-in Git integration (Source Control & Environments) only ships on **Enterprise** plans. If you're self-hosting on the free Community edition, the standard workflow is: export JSON → drag file into your repo folder → `git add` → `git commit` → `git push`, every single time you touch a workflow.

This project replaces that entire loop with one form and one click — n8n automating the versioning of its own workflows.

## ✨ Features

- **One-click sync** — pick a project from a dropdown, hit submit, done.
- **Live export, no manual steps** — pulls the current workflow JSON straight from n8n's REST API at the moment you sync.
- **Automatic create-or-update detection** — checks GitHub first, then creates a new file or updates the existing one. No manual SHA lookups.
- **Zero-noise commits** — a real byte-for-byte comparison means unchanged files never generate a commit. Re-running the sync on an untouched workflow produces nothing.
- **Multi-file uploads** — attach a README, screenshots, or any other file directly on the sync form. Files are routed automatically:
  - `README*` → project root
  - Images → `screenshots/` subfolder
  - Everything else → project root
- **Binary-safe** — correctly handles n8n's on-disk (`filesystem-v2`) binary storage mode, which silently breaks naive file-comparison logic if you don't account for it.
- **Fails loud, not quiet** — real failures (bad token, network issue) surface as visible errors instead of disappearing.

## 🧠 Engineering notes — the bugs that mattered

Three non-obvious issues came up building this, worth documenting for anyone extending it:

1. **GitHub's error output drops binary data.** When the GitHub node is set to continue on error, its error-output branch forwards only the JSON fields of the failed item — any attached binary is silently discarded. Fixed by re-attaching the original file's binary data from the upload step using `itemMatching()`, not by assuming the item still carries it.

2. **`filesystem-v2` binary storage isn't inline data.** Once n8n stores an uploaded file on disk (its default mode for anything beyond trivial size), the binary property's `data` field becomes a placeholder string (literally `"filesystem-v2"`) instead of actual base64 content. Comparing that placeholder against GitHub's real file content will *always* register as "different" — which is exactly what caused every sync to commit, even for identical files. The fix pulls the real bytes via n8n's `getBinaryDataBuffer()` helper before comparing.

3. **Index-based item matching breaks on mixed batches.** Matching "existing GitHub files" to "uploaded files" by array position silently mismatches results the moment you upload a batch containing both new and already-existing files (they don't stay in the same order once execution branches). Fixed with n8n's pairedItem-aware `itemMatching()` instead of naive array indexing.

## 🏗 Architecture

```
Form Trigger (select project, optionally attach files)
        │
        ├─▶ Get Workflow JSON ─▶ Check if file exists ─▶ Compare content ─▶ Update / Create (workflow.json)
        │
        └─▶ Split uploaded files ─▶ Check if file exists
                    ├─ exists ─▶ Reattach binary ─▶ Compare content ─▶ Update
                    └─ new    ─▶ Reattach binary ─▶ Create
```

## ⚙️ Setup

### Prerequisites
- Self-hosted n8n (Community edition is fine — no Enterprise features required)
- An n8n API key
- A GitHub Personal Access Token with `repo` scope

### 1. n8n API credential
Settings → **n8n API** → Create an API key.
Base URL: `https://<your-n8n-domain>/api/v1`

### 2. GitHub credential
[github.com/settings/tokens](https://github.com/settings/tokens) → Generate new token (classic) → check the `repo` scope.

### 3. Environment reference
See `.env.example`. Note: n8n stores credentials in its own encrypted vault, not a `.env` file — this is just a convenient place to stage the values before pasting them into the n8n credential UI.

### 4. Register your projects
Open the **Map Project** node and add one entry per project:

```js
your_workflow_name: {
  workflowId: 'the-n8n-workflow-id',
  repoFolder: 'the-github-folder-name'
}
```

Then add the matching option to the **Select Project** dropdown field on the form trigger.

## 🚀 Usage

1. Open the workflow in n8n and click **Test workflow** (or activate it) on the "Select Project" node.
2. Open the form URL n8n provides.
3. Pick a project. Optionally attach a README, screenshots, or other files.
4. Submit. Check your GitHub repo for the commit.

## ⚠️ Known limitations

- Project list is currently a manual map, not auto-discovered from the n8n API.
- Commits go straight to `main` — no branch/PR flow.
- Your local git clone can drift behind after a sync runs; `git pull` before continuing local work on the same repo.

## 📄 License

MIT
