# CFD Portfolio

Static site with three CFD case studies (race car external aero, supersonic
missile aero, Subaru Outback aero). Plain HTML/CSS, no build step.

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
index.html            Home page — links to all 3 case studies
racecarAero.html       Race car external aero case study
superSonic.html        Supersonic missile aero case study
subaruOutback.html     Subaru Outback aero case study
assets/css/style.css   Shared stylesheet (light/dark aware)
assets/img/            Curated, web-optimized result images (JPEG)
```

## Adding a new project page later

Copy an existing case-study HTML file as a starting template (they all share
the same section structure: header, headline stats, Problem & Goal,
Methodology, Engineering Challenges, Results), update its nav links, and add
a link to it from `index.html`'s project grid and every other page's nav bar.
