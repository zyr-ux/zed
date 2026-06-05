# Fork Changes

This document records the fork-specific changes in this repository so future merges from `upstream/main` can be resolved consistently.

It is based on:
- the `Fork Changes` section in `README.md`
- commits authored by `zyr-ux`
- the current diff between this fork and `upstream/main`

The goal is not just to list changed files, but to explain:
- what changed
- why it changed
- how it was implemented
- what should be preserved during future conflict resolution

---

## Source of truth

When merge conflicts happen, prefer this order of truth:
1. Keep upstream behavior by default.
2. Re-apply the fork-specific behavior described here where it still makes sense.
3. If upstream refactors the same code, preserve the intent of the fork change rather than blindly keeping old code.

The main fork-specific areas are:
1. Fork branding / logos
2. UI scrollbar styling
3. Replacing gradient-fade truncation with ellipsis truncation
4. Custom sync-and-release GitHub workflow
5. Updating the app updater to use this fork's GitHub releases
6. Showing fork release notes inside the app

---

## 1. Fork branding / logos

### Files
- `crates/zed/resources/app-icon.png`
- `crates/zed/resources/app-icon@2x.png`
- `crates/zed/resources/app-icon-dev.png`
- `crates/zed/resources/app-icon-dev@2x.png`
- `crates/zed/resources/app-icon-preview.png`
- `crates/zed/resources/app-icon-preview@2x.png`
- `crates/zed/resources/app-icon-nightly.png`
- `crates/zed/resources/app-icon-nightly@2x.png`
- `crates/zed/resources/windows/app-icon.ico`
- `crates/zed/resources/windows/app-icon-dev.ico`
- `crates/zed/resources/windows/app-icon-preview.ico`
- `crates/zed/resources/windows/app-icon-nightly.ico`

### Main commit
- `65d57555f1` — `updated all logos`

### What changed
The upstream app icons were replaced with fork-specific branding for all release channels and platforms currently represented in these resources.

### Why it was changed
This makes fork builds visibly distinct from upstream Zed. That matters for:
- user recognition
- avoiding confusion between upstream and fork binaries
- matching the custom release/update flow in this repository

### How it was implemented
The change is asset-only: the image/icon files in `crates/zed/resources/` were replaced.

### Merge guidance
- If upstream updates icon assets or adds new variants, keep the fork's branding unless you intentionally want to revert to upstream visuals.
- If upstream adds new release-channel icon files, create fork-branded equivalents for those too.
- Treat these files as fork-owned assets.

---

## 2. UI scrollbar styling

### File
- `crates/ui/src/components/scrollbar.rs`

### Main commits
- `c5f468852b` — `scrollbar: improved the scrollbar to be flush to the edge and removed rounded corners`
- `d0df5864b5` — `increased the ui scrollbar thickness`
- `1e2927b8e3` — `made the scrollbar a bit slimmer`
- `79aac88c44` — `improved the scrollbar padding`

### What changed
The non-editor UI scrollbar style (`ScrollbarStyle::Regular`) was customized.

Current fork behavior versus upstream:
- width is `8px` instead of `6px`
- rounded pill corners were removed; the thumb is square/flat
- padding was changed so the scrollbar sits flush against the outer edge instead of floating with symmetric padding
- inner spacing is preserved so the thumb still has breathing room toward the content side

### Why it was changed
The fork prefers a more solid, edge-aligned scrollbar appearance in UI surfaces outside the editor. The visual goals are:
- better visibility
- cleaner alignment to panel edges
- less of the floating/rounded look used upstream

### How it was implemented
In `crates/ui/src/components/scrollbar.rs`:
- `ScrollbarStyle::Regular` width was changed from `px(6.)` to `px(8.)`
- reserved track space for regular scrollbars was reduced from `2 * SCROLLBAR_PADDING` to `SCROLLBAR_PADDING`
- the thumb container bounds for regular scrollbars were changed from symmetric `dilate(-SCROLLBAR_PADDING)` to axis-specific edge extension logic
- rounded corners for regular thumbs were replaced with `Corners::default()`

### Merge guidance
If upstream changes scrollbar rendering:
- preserve the fork's `Regular` scrollbar width of `8px`
- preserve the edge-flush geometry for regular scrollbars
- do not reintroduce fully rounded regular scrollbar thumbs
- editor scrollbars should keep upstream behavior unless there is an intentional fork change

If upstream heavily refactors scrollbar layout, preserve the intent, not the exact old expressions:
- regular UI scrollbars should look flat, thicker, and flush to the outer edge
- editor scrollbars should remain separate from this customization

---

## 3. Replace gradient-fade truncation with ellipsis truncation

### Files
- `crates/agent_ui/src/agent_panel.rs`
- `crates/agent_ui/src/conversation_view/thread_view.rs`
- `crates/sidebar/src/sidebar.rs`
- `crates/ui/src/components/ai/thread_item.rs`

### Main commits
- `0e4a15a320` — `gradient_fade: removed the usage of the gradient fade truncation and replaced them with normal ellipses truncation`
- `89cb88ab88` — `removed more instances of gradient based truncation`
- `68103dd5d2` — `fixed the tool call truncation`
- `a85933d64e` — `fixed the thread title bar not having proper truncation`
- `6091ab9968` — `fixed the div() not having an id for editing the thread title`
- `c4413bb50a` — `updated the tool calls to use labels to properly use ellipsis truncation`
- `646c7438fe` — `reverted the tool calls truncation logic. it now matches the origianl logic`
- `e04460ccde` — `fixed the ai prompt editing controls clipping for the very 1st message in the thread`
- `8d48bb8dc0` — `fixed is_edit related warning`

### High-level intent
Upstream often used gradient fade overlays to visually soften truncated text. This fork prefers explicit ellipsis truncation (`...`) and layout spacing that keeps text readable without fade masks.

This is an important merge policy difference:
- upstream aesthetic: fade out overflowing text
- fork aesthetic: clamp/truncate text cleanly and make room for actions/controls

### Why it was changed
The fork prefers truncation that is:
- more explicit
- easier to read
- less visually muddy near action buttons
- more predictable when hover controls appear

### How it was implemented
The fork removed `GradientFade` usage in the affected views and replaced it with combinations of:
- `Label::truncate()`
- `HighlightedLabel::truncate()`
- `line_clamp(1)` where appropriate
- extra right padding when action buttons or counters appear
- absolute-positioned controls with solid backgrounds instead of faded overlays

### File-by-file notes

#### `crates/sidebar/src/sidebar.rs`

What changed:
- removed `GradientFade` import and overlay usage in sidebar headers
- added `.truncate()` to header labels and highlighted labels

Why:
- sidebar project/thread headers should end in a real ellipsis, not fade under controls

Merge guidance:
- do not reintroduce gradient overlays for sidebar header text
- keep explicit truncation on labels, even if upstream changes hover controls

#### `crates/ui/src/components/ai/thread_item.rs`

What changed:
- removed `GradientFade` import and overlay instances
- added `.truncate()` to thread title labels, including highlighted and title-generating cases
- when hover actions exist, right padding is added to the text row
- hover action slot is positioned absolutely with a solid background instead of fade masking

Why:
- thread titles should remain legible and truncate predictably when action buttons appear

Merge guidance:
- preserve label truncation
- preserve reserved space for hover actions
- do not restore gradient fade overlays around thread-item action buttons

#### `crates/agent_ui/src/agent_panel.rs`

What changed:
- removed toolbar/title `GradientFade`
- thread title display now uses a truncated `Label`
- added a clickable display container with ID `click-to-edit-thread-title`
- the title edit pencil now actively focuses the title editor
- added right padding when the title-edit affordance is visible so text does not sit under the button

Why:
- the title bar should truncate cleanly
- editing the title should be discoverable and reliable
- text should not visually collide with the edit control

Merge guidance:
- keep explicit truncation for the displayed title
- keep the click-to-edit wrapper and edit-button focus behavior
- if upstream redesigns the title bar, preserve the ability to click the title or pencil and enter edit mode without layout clipping

#### `crates/agent_ui/src/conversation_view/thread_view.rs`

What changed:
- removed gradient overlays from the pending-steps/status area and plan-entry rows
- added right padding when pending-count badges are visible
- added `line_clamp(1)` for plan-entry rows
- adjusted the edit-control overlay from `top_neg_3p5()` to `bottom_neg_3p5()` to avoid clipping on the first message
- there was an intermediate experiment around tool-call truncation, but the final state intentionally reverted tool-call logic closer to upstream while keeping surrounding layout/truncation fixes

Why:
- plan/status text should truncate with a visible ellipsis instead of fading out
- hover/edit controls should not cover text or get clipped
- the first prompt's edit controls needed a layout fix

Merge guidance:
- keep no-gradient truncation in this file
- keep spacing/clamping that prevents counters and controls from covering text
- preserve the first-message edit-control clipping fix
- if upstream changes tool-call rendering, do not blindly reapply any intermediate fork experiment; preserve the current behavior, which is closer to upstream in tool-call logic but still aligned with fork truncation/layout preferences

### Conflict-resolution rule for this whole area
If a future upstream merge reintroduces `GradientFade` into these UI surfaces, prefer the fork approach unless upstream has completely redesigned the view in a way that makes explicit ellipsis unnecessary.

In general, preserve:
- `.truncate()` on labels
- `line_clamp(1)` for single-line status rows
- explicit padding to reserve space for buttons/counters
- solid backgrounds or structural spacing instead of visual fade masks

---

## 4. Custom GitHub sync-and-release workflow

### File
- `.github/workflows/sync_and_release.yml`

### Main commits
- `9e78af9e14` — `added my own workflow file`
- `439e145186` — `updated workflow`
- `66ebed1a96` — `fixed workflow errors in the windows runner`
- `100564e176` — `updated the workflow and the auto updater to update zed from my custom fork builds`
- `90380008b1` — `fixed the long path error in the windows runner`
- `d41b7f5443` — `fixed workflow related to linux and mac runners`
- `97bb49eae9` — `updated the macos runner to macos 14 for reduced queue time when building`
- `37eef54d53` — `forced xcode version to 15.2/15.0.1 to fix the macos builds from failing`
- `386af9d64a` — `pinned cxx version in macos build to avoid xcode 15+ errors`
- `ac20a98ec3` — `fixed macos arm build issue`
- `5cf7c8298b` — `updated macos runners to use macos 15`
- `c9e9d4d805` — `improved workflow`
- `23d8541d56` — `updated workflow to to only sync the repo with the origianl repo daily but only release a build if a newer version is detected`
- `d290286e02` — `fixed a file lock issue in macos builds`
- `82b0ec33d6` — `fixed the paths for the various build binaries`
- `04063c4b79` — `improved the releases section of the builds`
- `4565bdf0d3` — `updated workflow to use github PAT`
- `8b60b3b417` — `updated release note generation through the workflow`

### What changed
This fork adds a repo-owned GitHub Actions workflow for syncing with `upstream/main`, building fork binaries, and publishing releases from this fork repository.

### Why it was changed
This fork needs its own release pipeline so users receive fork builds instead of upstream binaries, and so the internal updater can point at this repository's releases.

### Notes
Most of the workflow file is self-explanatory from the code. A number of the commits in this area were practical CI fixes for Windows, macOS, Linux, artifact paths, tokens, and release-note generation.

The important fork-level behavior to preserve is:
- releases are built and published from this fork repo, not upstream
- `ZED_UPDATE_REPO` is set to `${{ github.repository }}`
- scheduled syncs should not publish duplicate releases when the version already exists
- the published release notes should continue to include the upstream changelog context

### Merge guidance
Treat `.github/workflows/sync_and_release.yml` as fork-owned.

If upstream changes build scripts, artifact names, or release conventions:
- update this workflow to match the new reality
- keep it targeting the fork repository
- keep it aligned with the updater logic in `crates/auto_update/src/auto_update.rs`

---

## 5. Updater downloads from the fork's GitHub releases

### Files
- `crates/auto_update/src/auto_update.rs`
- `.github/workflows/sync_and_release.yml`

### Main commits
- `100564e176` — `updated the workflow and the auto updater to update zed from my custom fork builds`

### What changed
The app's auto-update logic was modified so it can download release assets from a configurable GitHub repository instead of assuming upstream's release endpoints.

### Why it was changed
Without this, the app would continue downloading official upstream binaries, which would bypass the fork's branding, UI changes, and release process.

### How it was implemented
In `crates/auto_update/src/auto_update.rs`:
- the updater reads `ZED_UPDATE_REPO` from the runtime environment first, then `option_env!`, and falls back to `zed-industries/zed`
- if the repo is not `zed-industries/zed`, the updater fetches release metadata from GitHub releases instead of using upstream's normal release asset path flow
- it resolves tags by release channel:
  - stable: `v{version}`
  - preview: `v{version}-pre`
  - nightly: `v{version}-nightly`
- if the channel-specific tag lookup fails, it falls back to plain `v{version}`
- it maps expected asset names to the workflow's artifact naming scheme:
  - macOS app: `Zed-{arch}.dmg`
  - Linux app: `zed-linux-{arch}.tar.gz`
  - Windows app: `Zed-{arch}.exe`
  - remote server: `zed-remote-server-{os}-{arch}.{gz|zip}`

The workflow sets:
- `ZED_UPDATE_REPO: ${{ github.repository }}`

That makes release publishing and app updates point to the same fork repository.

### Merge guidance
This is one of the most important fork-specific behaviors.

If upstream changes auto-update internals:
- preserve the `ZED_UPDATE_REPO` override path
- preserve GitHub-release lookup when repo != `zed-industries/zed`
- preserve channel tag naming and the fallback to plain `v{version}`
- preserve asset-name matching with whatever names the workflow actually uploads

If artifact names change in the workflow, update the auto-updater too. These two areas must stay in sync.

---

## 6. Show fork release notes inside the app

### Files
- `crates/auto_update/src/auto_update.rs`
- `crates/auto_update_ui/src/auto_update_ui.rs`
- `crates/http_client/src/github.rs`

### Main commits
- `b110b407fb` — `added support for changelog dsiplay in the app for the custom fork releases`

### What changed
When the app is configured to update from this fork's repository, the in-app "View Release Notes" flow fetches release notes from the fork's GitHub release body and displays them locally.

### Why it was changed
Upstream release-note fetching expects Zed's own backend/API. For fork releases, the authoritative release notes live in GitHub Releases for this repository.

### How it was implemented
#### `crates/http_client/src/github.rs`
- `GithubRelease` now includes `body: Option<String>`
- added `get_release_by_tag_name(...)` to fetch a specific release by tag

#### `crates/auto_update/src/auto_update.rs`
- added `fetch_github_release_notes(repo, tag, client)`
- this fetches the tagged GitHub release and returns `(title, body)`

#### `crates/auto_update_ui/src/auto_update_ui.rs`
- `view_release_notes_locally(...)` now checks `ZED_UPDATE_REPO`
- if the repo is not upstream, it fetches release notes from GitHub instead of using the Zed API
- it tries the channel-specific tag first and falls back to plain `v{version}`
- it opens the fetched markdown in an in-app markdown preview buffer
- if the fetch fails, it falls back to the existing error-notification flow

### Merge guidance
If upstream changes release-note UI:
- preserve the fork path that reads from GitHub release bodies when `ZED_UPDATE_REPO` points away from upstream
- preserve fallback from channel-specific tags to plain `v{version}`
- preserve local markdown preview rendering for fetched notes

If upstream adds a new generic abstraction for release-note providers, this fork behavior should move into that abstraction rather than being deleted.

---

## 7. README changes

### File
- `README.md`

### Main commits
- `d5a64d1f7c` — `updated Readme`
- `06775981b3` — `updated readme`

### What changed
The README was updated to:
- rename the title to `Zed - Fork`
- add a `Fork Changes` section summarizing the fork-specific differences

### Why it was changed
This makes the repository self-describing and helps future maintainers quickly identify intentional divergence from upstream.

### Merge guidance
If upstream rewrites the README, keep a short fork-specific section near the top or otherwise keep a clear pointer to `docs/src/fork-changes.md`.

---

## Merge checklist

When resolving future conflicts, check these points first:

### Branding
- Are fork logo assets still present for all release channels?

### Scrollbars
- Is `ScrollbarStyle::Regular` still thicker than upstream?
- Is the thumb still square/flat, not rounded?
- Is it still flush to the outer edge?

### Truncation / text overflow
- Were any `GradientFade` overlays reintroduced in:
  - `crates/agent_ui/src/agent_panel.rs`
  - `crates/agent_ui/src/conversation_view/thread_view.rs`
  - `crates/sidebar/src/sidebar.rs`
  - `crates/ui/src/components/ai/thread_item.rs`
- Do labels still use explicit truncation/clamping?
- Do action buttons/counters still reserve layout space instead of covering text?

### Releases / updater
- Does `.github/workflows/sync_and_release.yml` still publish releases in the fork repo?
- Does it still set `ZED_UPDATE_REPO`?
- Does `crates/auto_update/src/auto_update.rs` still honor `ZED_UPDATE_REPO`?
- Do workflow artifact names still match updater asset-name expectations?
- Does release-note viewing still work for fork releases via GitHub release bodies?

---

## Practical conflict strategy

If a future merge conflicts in one of these files:

1. Read the upstream change and determine whether it is:
   - a functional bug fix
   - a refactor
   - a visual redesign
   - a release pipeline change
2. Preserve the upstream improvement where possible.
3. Re-apply the fork intent from this document.
4. Verify the fork-specific behavior still exists after the merge.

Examples:
- If upstream rewrites scrollbar rendering, keep the new rendering architecture but re-apply the fork's flat, thicker, edge-flush regular scrollbar style.
- If upstream rewrites title bars or thread items, keep the new structure but prefer explicit ellipsis truncation over fade masks.
- If upstream changes release artifact names, update both the workflow and the updater's asset lookup together.
- If upstream changes release-note fetching, keep the fork's GitHub-release-body path for non-upstream repositories.

---

## AI agent merge policy

For merge conflicts that touch fork-specific behavior described in this document, an AI agent should not silently finalize the resolution on its own.

Expected process:
1. Identify the upstream change.
2. Identify the conflicting fork-specific behavior.
3. Resolve the conflict in a proposed way that preserves the intended fork behavior where appropriate.
4. Explain the resolution to the fork owner before finalizing the code edits.
5. Wait for approval from the fork owner before treating the merge resolution as final.

The explanation to the fork owner should include:
- what upstream changed
- what the fork had changed previously
- what differs between the two
- how the proposed merge resolution handles that difference
- whether the result stays closer to upstream, closer to the fork, or is a hybrid of both

In short: for fork-sensitive merge conflicts, propose first, explain clearly, then finalize only after approval.

This policy is especially important for conflicts involving:
- `.github/workflows/sync_and_release.yml`
- `crates/auto_update/src/auto_update.rs`
- `crates/auto_update_ui/src/auto_update_ui.rs`
- `crates/http_client/src/github.rs`
- `crates/ui/src/components/scrollbar.rs`
- `crates/agent_ui/src/agent_panel.rs`
- `crates/agent_ui/src/conversation_view/thread_view.rs`
- `crates/sidebar/src/sidebar.rs`
- `crates/ui/src/components/ai/thread_item.rs`
- fork branding assets in `crates/zed/resources/`

---

## Summary

This fork intentionally diverges from upstream in four main ways:
- fork branding
- a custom scrollbar style
- explicit ellipsis-based truncation instead of gradient fades in several UI surfaces
- a fully fork-owned release/update pipeline, including in-app release notes

When in doubt during merges, preserve upstream structure and bug fixes, but keep these fork-level product decisions intact unless there is an explicit decision to drop them.
