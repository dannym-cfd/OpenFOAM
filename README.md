# CFD Portfolio

Static site with three CFD case studies (race car external aero, supersonic
missile aero, Subaru Outback aero). Plain HTML/CSS, no build step.

<p>
  <img src="assets/img/racecarAero-streamline-render.jpg" alt="Streamline visualization over the race car, colored by surface pressure, on a black background" width="49%">
  <img src="assets/img/missile-shock-render.jpg" alt="Schlieren-style rendering of the missile in supersonic flight showing the bow shock cone" width="49%">
</p>

## Publishing to GitHub Pages (free)

1. Repository already created: `https://github.com/dannym-cfd/OpenFOAM`.
2. From this folder, push it to that repo:
   ```
   git init
   git add .
   git commit -m "Initial CFD portfolio site"
   git branch -M main
   git remote add origin https://github.com/dannym-cfd/OpenFOAM.git
   git push -u origin main
   ```
3. In the GitHub repo, go to **Settings &rarr; Pages**.
4. Under "Build and deployment" &rarr; Source, choose **Deploy from a
   branch**, then set Branch to `main` and folder to `/ (root)`. Save.
5. GitHub will publish the site at `https://dannym-cfd.github.io/OpenFOAM/`
   within a minute or two — refresh the Pages settings page to get the exact
   URL once it's live.

## Updating later

Edit the HTML/CSS files directly, then:
```
git add .
git commit -m "Update case study content"
git push
```
GitHub Pages redeploys automatically on every push to `main`.

## Structure

```
index.html                Home page — links to all 3 case studies
racecarAero/index.html    Race car external aero case study
superSonic/index.html     Supersonic missile aero case study
subaruOutback/index.html  Subaru Outback aero case study
assets/css/style.css      Shared stylesheet (light/dark aware)
assets/img/               Curated, web-optimized result images (JPEG)
```

Each case study lives in its own folder as `index.html` rather than a flat
`name.html` file, so GitHub Pages serves it at a clean folder URL (e.g.
`/racecarAero/`) and the repo's file browser shows a folder instead of a raw
`.html` file.

## Adding a new project page later

Copy an existing case-study folder (e.g. `racecarAero/`) as a starting
template — they all share the same `index.html` section structure (header,
headline stats, Problem & Goal, Methodology, Engineering Challenges,
Results). Inside the new folder, update its nav links and the `../` asset
paths, then add a link to it from `index.html`'s project grid and every
other page's nav bar.
