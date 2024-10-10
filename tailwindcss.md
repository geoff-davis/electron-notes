# Tailwind CSS notes

[Tailwind CSS](https://tailwindcss.com/) provides useful starting point for your CSS, and several people in [r/electronjs](https://www.reddit.com/r/electronjs/) recommend it. There are probably lots of other good options.

## Setting up Tailwind CSS

```bash
cd myappname
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Configuration

Modify the electron-vite default `tailwind.config.js` file to pick up renderer files via the `content` property. The new config file:

```javascript
/** @type {import('tailwindcss').Config} */

module.exports = {
  content: ['./src/renderer/index.html', './src/renderer/src/*.{vue,js,ts,jsx,tsx}'],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

### VSCode integration

Add the [Tailwind CSS IntelliSense](https://marketplace.visualstudio.com/items?itemName=bradlc.vscode-tailwindcss) plugin to VSCode.

```bash
cd myappname
npm install -D prettier prettier-plugin-tailwindcss
```

Change the following settings in VSCode:

```javascript
"files.associations": {
    "*.css": "tailwindcss"
}
```

```js
"editor.quickSuggestions": {
    "strings": "on"
}
```

Add the following to `.prettierrc.yaml`:

```yaml
plugins:
  - prettier-plugin-tailwindcss
```

### Basic styling

ChatGPT suggested a Tailwind theme that resembled a MacOS-native app.
Here's the updated `tailwind.config.js`:

```javascript
/** @type {import('tailwindcss').Config} */

module.exports = {
  content: ['./src/renderer/index.html', './src/renderer/src/*.{vue,js,ts,jsx,tsx}'],
  // macos native theme from chatgpt
  theme: {
    extend: {
      fontSize: {
        root: ['13px', '1.5'],
      },
      fontFamily: {
        sans: [
          '-apple-system',
          'BlinkMacSystemFont',
          '"Segoe UI"',
          'Roboto',
          'Oxygen',
          'Ubuntu',
          'Cantarell',
          '"Open Sans"',
          '"Helvetica Neue"',
          'sans-serif',
        ],
      },
      colors: {
        mac: {
          gray: {
            light: '#F5F5F7',
            DEFAULT: '#D1D1D6',
            dark: '#636366',
          },
          blue: '#007AFF',
          red: '#FF3B30',
          green: '#34C759',
          yellow: '#FFCC00',
        },
        macButton: {
          DEFAULT: '#F0F0F5',
          hover: '#E5E5EA',
          active: '#D9D9DE',
        },
      },
      borderRadius: {
        sm: '4px',
        md: '6px',
        lg: '12px',
      },
      boxShadow: {
        mac: '0 4px 8px rgba(0, 0, 0, 0.05)',
        macHover: '0 6px 12px rgba(0, 0, 0, 0.1)',
      },
      spacing: {
        1.5: '6px',
        2.5: '10px',
        7: '28px',
      },
    },
  },
  plugins: [],
};
```

Here is the new `renderer/assets/main.css` file:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Apply the root font size from tailwind.config.js */
html {
    @apply text-root;
}
```
