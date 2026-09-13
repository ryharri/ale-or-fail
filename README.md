# Ale or Fail

A dark, iPhone-styled Progressive Web App for logging ale and lager ratings. Ratings are stored only in the browser's local storage, so the app works privately and offline after its first visit.

## Run locally

```sh
npm install
npm run dev
```

## Publish free with GitHub Pages

1. Create a GitHub repository named `ale-or-fail` and push this folder to its `main` branch.
2. In GitHub, open **Settings → Pages** and set the source to **GitHub Actions**.
3. The included deployment workflow publishes every push to `main`.

The Vite configuration already uses `/ale-or-fail/` as the production path. If your GitHub repository has a different name, update that path in `vite.config.js`.

## Install on iPhone

Open the deployed site in Safari, tap Share, then choose **Add to Home Screen**. It opens as a standalone app and retains ratings locally on that iPhone.
