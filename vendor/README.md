# vendor/ — libraries for fully offline use

By default, `index.html` pulls its JS dependencies from jsDelivr so the app
works with zero build step on any modern browser. If you want to run the
dashboard without a network connection (from a USB drive, a lab workstation
with blocked egress, an air-gapped environment), drop local copies of the
libraries into this folder and flip the `<script src="…">` tags in
`index.html` to point at them.

## One-shot download

```sh
cd vendor
curl -LO https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js
curl -LO https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js
curl -LO https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js
curl -LO https://cdn.jsdelivr.net/npm/jspdf@2.5.1/dist/jspdf.umd.min.js
```

After downloading, change the four `<script src="https://cdn.jsdelivr.net/...">`
tags near the end of `<body>` in `index.html` to the local paths:

```html
<script src="vendor/papaparse.min.js"></script>
<script src="vendor/chart.umd.min.js"></script>
<script src="vendor/html2canvas.min.js"></script>
<script src="vendor/jspdf.umd.min.js"></script>
```

Google Fonts (Inter, JetBrains Mono) are loaded the same way from
`fonts.googleapis.com`. For a truly offline build, either self-host the two
font families in this folder and swap the `<link rel="stylesheet">` to
reference them, or let the browser fall back to the system sans-serif /
monospace declared in the `--font-sans` / `--font-mono` custom properties.

## Why this isn't already inlined

Chart.js (~200 KB), html2canvas (~200 KB), and jsPDF (~350 KB) add ~1 MB to
`index.html`, making the single-file artifact noticeably heavier to load over
the network or inspect in a text editor. Keeping the libs external is the
right default for the hosted web version; the vendor-local path is the
right default for offline and air-gapped deployments.
