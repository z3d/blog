# Developer notes

Static GitHub Pages site for developer notes by z3d.

Live site: <https://z3d.github.io/blog/>

## Posts

- [Measuring consistency in agent-generated code](site/consistency-checks/)

## Development

Preview locally:

```bash
python3 -m http.server 4173 --directory site
```

Then open <http://127.0.0.1:4173/>.

## Analytics

Canonical pages include the GoatCounter snippet for `z3d.goatcounter.com`.
Redirect-only pages intentionally omit analytics to avoid double-counting legacy URLs.

Create the GoatCounter account with the `z3d` site code before publishing, or update the
snippet if that code is already taken.
