# IQIQ LLC — working notes

IQIQ LLC — a holding page for the company, one static index.html with a canvas starfield and nebula behind it.

Served at **https://iqiq.llc** from the lab980 droplet.

How work lands here — branch, PR, and the fact that merging is not deploying —
is in `.claude/rules/lab980-conventions.md`, which Claude Code loads
automatically every session. That file is owned by the lab980 scaffold and is
overwritten by it; **this** file is the site's own, and everything below is
about this site rather than about the platform. For the box itself, read the
`ivjames/lab980.com` repo's `CLAUDE.md`.

## Shape

Fully **static**: the site is files served straight by nginx. No build step,
no app process, no local port, no pm2, no database. nginx serving the git
checkout *is* the deployment, so "what's on `main`" and "what's live" differ
only by a `git reset` on the droplet.

- Repo: `ivjames/iqiqllc` · droplet checkout: `/var/www/iqiq` (the web root)
- **Apex domain**, not a `*.lab980.com` subdomain: `iqiq.llc` plus a `www`
  alias, the way `boxoffice` (`boxo.show`) and `vinaigresque` do it. `iqiq
  setup` creates both A records in the `iqiq.llc` DigitalOcean zone (record
  `@` and `www`), the vhost names both in one server block, and certbot is
  asked for one cert covering both. No www -> apex redirect. `--no-www` or
  `IQIQ_WWW=0` drops the alias everywhere at once.
- The DNS zone `iqiq.llc` has to exist in DigitalOcean before `setup`
  (`doctl compute domain create iqiq.llc`), and the registrar's nameservers
  have to point at DigitalOcean — neither is something this repo can do.
- Operate CLI: `bin/iqiq`, symlinked to `/usr/local/bin/iqiq`
- vhost: generated from `deploy/nginx.conf.template` by `iqiq setup`

## Deploying

On the droplet, as root:

```bash
iqiq deploy      # git fetch + reset --hard origin/main (+ build stamp)
iqiq status      # HEAD, live probe, cert days remaining
```

Full runbook, including first-time bring-up: `DEPLOY.md`.

Checking what is actually live, concretely for this site — `iqiq status`
on the box, or from anywhere:

```bash
curl -s -o /dev/null -w 'HTTP %{http_code}\n' https://iqiq.llc/
curl -s https://iqiq.llc/ | grep -o "const BUILD = '[^']*'" | head -1
```

(The second line reports nothing if the page carries no `BUILD` constant — see
the deploy stamp note in `DEPLOY.md`. `head -1` because a page that polls its
own build stamp carries a matching regex literal, which grep otherwise reports
as a phantom second build.)

## The page

One `index.html`, no dependencies, no fetches. A `<canvas>` behind the text
paints a nebula once to an offscreen canvas (two value-noise/fBm fields, one
warping the other, tinting five soft colour clouds with dark dust lanes) and
then draws a dense starfield over it every frame — three depth layers drifting
at different speeds, a few percent of stars bright with a halo, some twinkling.
Everything is seeded (`mulberry32`), so the sky is the same on every visit.
`prefers-reduced-motion` renders the same sky once and skips the animation
loop; the loop also pauses while the tab is hidden. A **Pause sky / Play sky**
button in the footer (WCAG 2.2.2) stops and restarts the drift and twinkle;
under reduced motion it starts as "Play sky", offering the animation as an
opt-in. Its label is the action and it deliberately carries no `aria-pressed`:
that attribute belongs with a stable name, and "Play sky, pressed" would
contradict itself.

Text sits over a starfield, so every text element carries two dark
`text-shadow`s (the `--halo` token): a thick near-solid one hugging the
letterforms (zero-blur offset copies at 1px and 2px in eight directions plus 3px
in the four cardinal ones, a near-circular solid ring, carried out smoothly by
stacked short blurs) and a wide soft one dimming the sky around the block, so a star
landing under a glyph does not steal its contrast. Secondary text and the
footer are at 72% opacity for the same reason: at 55% the best possible ratio
was 5.3:1 with no headroom, and the old 30% footer failed the 4.5:1 AA floor
outright. Measured under and 1px around every glyph with the text rendered
transparent, against the real sky and against an all-white canvas.

The page carries `const BUILD = 'dev';` at two-space indent, which `iqiq
deploy` stamps with the deployed short SHA; the footer shows it once stamped
and stays blank on a `dev` checkout.

## Things worth knowing

- The droplet checkout is the web root, so anything committed here is public
  except dotfiles and `*.md` (the vhost denies both; the dotfile deny is the
  `(?!well-known)` form so certbot's HTTP-01 challenge path stays reachable).
  Don't commit secrets; there is no `.env` on a static site.
- There is no `.env` here and nothing to keep out of git beyond that — a
  static site has no secrets to hold.

## Co-packer memory docs

Correspondence with co-packers is tracked in one Claude Doc per co-packer.
Those docs are the memory; a session's own transcript is not, because a new
session starts without it. This applies to every session in this repo,
whatever it was opened for.

**The repo has to be mounted for any of this to apply.** This file loads only
in a session that has `ivjames/vinaigresque` or `ivjames/iqiqllc` attached as
a repository; both carry this section. Start every co-packer session with one
of them attached, and when a session spawns another (for example with
`create_session`), pass the repo as its source. A session started with no repo
never sees this file, so its first message has to carry the doc link and these
rules instead.

- **Before** discussing a co-packer, read its doc with the Claude Docs tools.
- **Whenever** a co-packer fact turns up (an email, a call, a quote, a
  packet, a decision about one), write it to that co-packer's doc before
  ending the turn: Terms, Open questions and next step, and a Correspondence
  log line (newest first). Facts only, each with its source; mark anything
  unconfirmed. Do this even if the session was opened for something else.
- **A co-packer with no doc yet**: create one in the same shape (How to use,
  Company facts, Terms, Open questions and next step, Correspondence log)
  and add it to the table below in both repos' `CLAUDE.md`, which carry the
  same section and must stay in step.
- If the Claude Docs tools are unavailable, say so and give Arthur the facts
  to paste in; don't let them live only in the chat.

The docs hold the recipe weights and quotes. They stay in Claude Docs, never
in this repo: the checkout is the public web root.

| Co-packer | Format | Memory doc |
| --- | --- | --- |
| Ibitta Enterprises | Pouch (toll) | https://claude.ai/code/artifact/830a7ccb-d9aa-47fd-bed9-dc219504abb9 |
| Co-Packing Express | Pouch | https://claude.ai/code/artifact/2928e9b4-2771-4524-a3fd-a4c666211130 |
| Qualia Provisions | Pouch | https://claude.ai/code/artifact/e88af225-7a66-4631-9cec-6c357c8de75d |
| Vanns Spices | Pouch | https://claude.ai/code/artifact/a56f661c-69b8-4f80-94d7-8961c62dc3a0 |
| Pacific Spice Company | Unknown | https://claude.ai/code/artifact/dd17e2b5-7bf8-4ee0-b4d8-17d8f78ce925 |
| The Spice Guy | Jar | https://claude.ai/code/artifact/6f17200b-d0ad-403b-8112-2f38958a8323 |
| The Spice Lab | Jar | https://claude.ai/code/artifact/2e692d62-1d48-4259-a37b-e69b68668eb8 |
| Tampico Spice | Jar | https://claude.ai/code/artifact/4bcfe71c-2dbf-4b99-bcac-d1dd542cd2b0 |
| La Criolla | Jar | https://claude.ai/code/artifact/7dd6e090-61cd-4e24-86fd-93d2623428f1 |
