# iiif-tools

Web tool designed for the Leventhal Map & Education Center to access various IIIF endpoints, and extract image crops and extent coordinate arrays.

`iiif-tools` will theoretically work with any IIIF Manifest, but it's designed to consume LMEC Digital Collections and Digital Commonwealth URLs: you can paste those directly into the input. The app is built in Vue 3 (updated from Vue 2 in March 2026, with some help from Claude). It uses OpenLayers for manipulating and cropping IIIF images in a canvas.

In action: <https://iiif-tools.leventhal.center>

# Build and deploy

To build and develop:

1. clone this repository
2. `cd` into it
3. `npm install`
4. `npm run dev`

The app continuously deploys from Netlify, via the `main` branch of this repository. If you want to stage edits or test changes, make a `staging` branch or some such.