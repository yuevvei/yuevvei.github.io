# Marketing and legal site (courtcamper.com)

Guidance for this directory, split out of the root `CLAUDE.md`. Loaded only when Claude works with files here.

Plain hand-written HTML with no build step, no framework, and no shared stylesheet — each page carries its own inline `<style>` block, copied from `privacy.html` and extended where a page needs something extra. When adding a page, start from `privacy.html`'s head so the nav, footer, type scale, and `--primary: #FF5722` tokens match.

## Pages, and what depends on them

Every file here is load-bearing for something outside the repo. **Do not rename or delete one without changing the thing that points at it.**

| File | Depended on by |
|------|----------------|
| `index.html` | Landing page. Links to both app stores. |
| `privacy.html` | The privacy policy URL in **both** the App Store listing and Play Console. |
| `delete-account.html` | The account-deletion URL declared in Play Console → Data safety. Play requires a deletion path reachable **without** installing the app. |
| `get.html` | Every in-app invite message links here — see `buildInviteMessage()` in `lib/utils/invite_message.dart`. Deliberately not a store link: whoever copies an invite has no idea which phone the recipient carries. |
| `confirm.html` | Supabase auth email-confirmation redirect. |
| `reset-password.html` | Supabase auth password-reset redirect, registered in Supabase Dashboard → Authentication → URL Configuration. |
| `CNAME` | Binds `courtcamper.com` to the Pages repo. |

`delete-account.html` documents what `delete_user_account()` actually does — stats preserved with the account link removed, messages de-attributed, sole-staff deletion blocked, head coach transferred. If that RPC's behaviour changes, this page becomes a false statement to a store reviewer, so change both together.

## Deployment

`.github/workflows/deploy-site.yml` publishes this directory on every push to `main` that touches `site/**`, plus `workflow_dispatch` for a manual republish.

**It does not deploy to this repo's own GitHub Pages.** It mirrors `site/` into a separate public repo, `yuevvei/yuevvei.github.io`, which is what actually serves `courtcamper.com`.

**Why the indirection:** this repo is private and the account is on GitHub Free, where Pages only works on public repos. Making this repo public is not the escape hatch either — `lib/config.dart` is committed with real API keys and would remain in the git history after any fix. So the domain stays on the public repo and CI does the copying.

Before this workflow existed the copy was manual, and it had drifted: the live privacy policy sat five months behind the one in this repo. **Edit `site/` and push — never edit the Pages repo directly**, or the drift comes straight back.

### Mirror semantics

The job runs `rsync -a --delete`, so it is an exact mirror: a file deleted from `site/` is deleted from the live site. The Pages repo holds nothing but this site, which is what makes that safe. Two guards, because it deletes files in a repo the job does not own:

- **`site/CNAME` must exist** or the run fails before touching anything. Mirroring without it would delete `CNAME` at the target and quietly unbind the custom domain — a failure that looks like the site vanishing.
- **`.git` is both `--exclude`d and explicitly `--filter 'protect'`ed.** The exclude alone already shields the destination from `--delete` (hence `--delete-excluded` existing as a separate flag), but the redundancy is cheap next to deleting another repo's `.git`.

The `Publish if anything changed` step prints `git status --short` before committing. On any run that changes the site, read it: adds and modifications are expected, and a `D` line means a file exists at the target that `site/` does not know about.

Nothing here is unrecoverable — the target is a git repo with full history, so a bad mirror is one revert away.

### Auth

A **deploy key**, not a personal access token: it grants write to `yuevvei.github.io` only, so a leak cannot reach the rest of the account. The public half is a deploy key (with write access) on the Pages repo; the private half is the `SITE_DEPLOY_KEY` Actions secret on this repo. The workflow writes it to `~/.ssh/id_ed25519` on the runner at job start.

To rotate: `ssh-keygen -t ed25519`, replace the deploy key on the Pages repo and the secret here. Nothing else in the setup changes.
