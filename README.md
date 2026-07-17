# Split Shot

Split Shot is a Svelte game where you move a gun vertically, shoot bouncing balls, and advance through increasingly difficult rounds.

Controls:

- Arrow Up and Arrow Down: move the gun
- Space: shoot

Desktop only for now.

## Local development

Install dependencies:

```sh
npm install
```

Run the dev server:

```sh
npm run dev
```

Create a production build:

```sh
npm run build
```

Preview the production build locally:

```sh
npm run preview
```

## GitHub Pages deployment

This repository includes a GitHub Actions workflow at .github/workflows/deploy-pages.yml that builds and deploys on every push to main.

Required GitHub settings:

1. Open repository Settings -> Pages.
2. In Build and deployment, set Source to GitHub Actions.
3. Push to main.
4. Watch the Actions tab for the Deploy to GitHub Pages workflow.
5. After success, your site will be available at your repository Pages URL.

Notes:

- The static output folder is build.
- The workflow adds build/.nojekyll automatically.
