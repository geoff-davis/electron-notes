# electron-vite notes

[electron-vite](https://electron-vite.org/) is a great way to get set up for writing electron apps.

The package gives you:

* an [electron](https://electronjs.org) instance
* [vite](https://vitejs.dev) for bundling js and letting you do hot-reloading while developing your app
* basic scaffolding for an electron app in your choice of
  * Front-ends: vanilla / Vue / React / Svelte / Solid
  * Javascript: plain JS or TypeScript
* some useful VSCode configuration

## Setting up electron-vite

electron-vite sets up a 2-level directory structure. The outer, top-level directory contains electron and various dependencies; the inner directory contains your project and code.

Here we'll name the outer directory `myproject` and the inner directory `myappname`:

```bash
mkdir myproject
cd myproject
npm install -D electron-vite
npm create @quick-start/electron@latest
# At this point you will be prompted to specify your app name
# (myappname) and the type of scaffolding you want

cd myappname
npm install
```

## Setting up VSCode with electron-vite

If you're using VSCode, create a new workspace in the *inner* folder (`myproject/myappname`).

### Useful VSCode extensions

* [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)
* [Npm Intellisense](https://marketplace.visualstudio.com/items?itemName=christian-kohler.npm-intellisense)
* [Prettier - Code formatter](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode)
* [Prettier ESLint](https://marketplace.visualstudio.com/items?itemName=rvest.vs-code-prettier-eslint)

### Tweaking prettier

A few changes to the stock electron-vite `.prettierrc.yaml`:

```yaml
singleQuote: true
semi: true
trailingComma: all
```
