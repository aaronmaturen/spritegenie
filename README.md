# SpriteGenie 🧞‍♂️

Transform sprite sheets into animated GIFs with magical ease! SpriteGenie is a web-based tool that brings your pixel art to life with powerful features like automatic frame detection, jitter reduction, and bidirectional conversion between sprite sheets and GIFs.

## ✨ Features

- **Sprite Sheet → Animated GIF**: Convert sprite sheets into smooth animated GIFs
- **Animated GIF → Sprite Sheet**: Extract frames from GIFs and arrange them as sprite sheets
- **Automatic Frame Detection**: Smart grid detection finds sprite boundaries automatically
- **Border Auto-Cropping**: Removes unnecessary transparent borders with visual overlay
- **Jitter Reduction**: Center frames to eliminate animation jitter
- **Gap Support**: Handle sprite sheets with spacing between frames
- **Real-time Preview**: See your animation before exporting
- **Customization Options**:
  - Adjustable frame dimensions
  - Variable animation speed (FPS)
  - Custom background colors
  - Manual crop controls
  - Row/column gap settings

## 🚀 Quick Start

### Development

```bash
# Install dependencies
npm install

# Start development server
npm run dev
```

### Build

```bash
# Build for production
npm run build

# Preview production build
npm run preview
```

## 🎮 Usage

1. **Upload** your sprite sheet or animated GIF
2. **Adjust** frame dimensions and grid settings (or let auto-detection handle it)
3. **Preview** your animation in real-time
4. **Customize** animation speed, background, and other settings
5. **Generate** and download your animated GIF or sprite sheet

## 🛠️ Tech Stack

- **Vue 3** - Progressive JavaScript framework
- **Vite** - Fast build tool and dev server
- **gif.js** - GIF encoding in JavaScript
- **gifuct-js** - GIF parsing and frame extraction

## 📦 Project Structure

```
sprite/
├── src/
│   ├── App.vue          # Main application component
│   └── main.js          # Application entry point
├── public/
│   ├── favicon.png      # Genie favicon
│   ├── CNAME           # Custom domain configuration
│   └── gif.worker.js   # Web Worker for GIF generation
├── package.json         # Project dependencies
├── vite.config.js      # Vite configuration
└── index.html          # HTML entry point
```

## 🌐 Deployment

The project includes GitHub Actions for automatic deployment to GitHub Pages:

```yaml
# Pushes to main branch trigger automatic deployment
git push origin main
```

The site is deployed to: [spritegenie.aaronmaturen.com](https://spritegenie.aaronmaturen.com)

## 🎯 Use Cases

- Game developers creating animated sprites
- Pixel artists showcasing animations
- UI/UX designers working with animated icons
- Converting between sprite sheets and GIF formats
- Creating smooth animations from sprite sheets with gaps

## 🤝 Contributing

Contributions are welcome! Feel free to:
- Report bugs
- Suggest new features
- Submit pull requests

## 📝 License

This project is open source and available under the MIT License.

## 👨‍💻 Author

Created by [Aaron T. Maturen](https://aaronmaturen.com) as part of the [digital follies](https://aaronmaturen.com/follies) collection.

---

*SpriteGenie - Where sprite sheets come to life! ✨*