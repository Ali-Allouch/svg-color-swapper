# SVG Color Swapper 🎨

![Version](https://img.shields.io/badge/version-1.0.1-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Vue.js](https://img.shields.io/badge/Vue.js-3.x-4fc08d?logo=vue.js)
![TypeScript](https://img.shields.io/badge/TypeScript-6.x-3178c6?logo=typescript)
![Vite](https://img.shields.io/badge/Vite-8.x-646CFF?logo=vite)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-v4-06B6D4?logo=tailwindcss)

**SVG Color Swapper** is a professional Chrome Extension designed for developers and designers. It allows you to instantly detect, live-edit, and export SVG icons from any webpage with a few clicks.

---

## ✨ Key Features

- **Smart SVG Detection:** Automatically finds all SVG elements on a page, handling complex scenarios like `currentColor`, `rgba` colors, and inherited CSS styles.
- **Live Color Swapping:** Modify SVG colors (Fill & Stroke) and see the changes reflected instantly on the original website.
- **Clean Export:** Export modified SVGs with "Baking" logic that removes technical attributes and inlines styles for a production-ready file.
- **Developer-Friendly UI:** A sleek, dark-themed interface built with **Vue 3** and **Tailwind CSS** for a seamless workflow.
- **Privacy First:** 100% local processing. No data ever leaves your browser.

---

## 🛠️ Tech Stack

- **Frontend:** [Vue 3](https://vuejs.org/) (Composition API)
- **Bundler:** [Vite](https://vitejs.dev/)
- **Styling:** [Tailwind CSS](https://tailwindcss.com/)
- **Language:** [TypeScript](https://www.typescriptlang.org/)
- **API:** Chrome Extensions API (Manifest V3)

---

## 🚀 Installation for Developers

1. **Clone the repository:**
   ```bash
   git clone https://github.com/Ali-Allouch/svg-color-swapper.git
   cd svg-color-swapper
   ``` 
2. **Install dependencies:**
   ```bash
   npm install
   ```

3. **Build the extension:**
   ```bash
   npm run build
   ```

4.  **Load into Chrome as an unpacked extension**:
    -  Open Chrome and navigate to `chrome://extensions`.
    -  Enable "Developer mode" by toggling the switch in the top right corner.
    -  Click on "Load unpacked" and select the `dist` folder generated in the previous step.

## 📂 Project Structure

```
.gitignore
index.html
LICENSE
manifest.json
package-lock.json
package.json
PRIVACY.md
README.md
tsconfig.app.json
tsconfig.json
tsconfig.node.json
vite.config.ts
public/
└── favicon/
src/
├── App.vue
├── main.ts
└── style.css
```

## 📄 Privacy Policy

We value your privacy. This extension does not collect or transmit any user data. Read the full [Privacy Policy](PRIVACY.md) for more details.

---

## ⚖️ License

Distributed under the MIT License. See `LICENSE` for more information.

---
*Developed with focus on scalability and clean code by [**Ali Allouche**](https://aliallouche.me/).*