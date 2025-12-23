# Paste List · Vue 3 Interactive Checklist

> A quick start project for creating an interactive checklist where you can paste text and check off items.

## Table of Contents

- [Features](#features)
- [Technologies](#technologies)
- [Installation](#installation)
- [Running the App](#running-the-app)
- [Project Structure](#project-structure)
- [Future Enhancements](#future-enhancements)
- [License](#license)

## Features

- Manage multiple checklists with custom titles
- Paste a list of items (each line becomes a separate item)
- Interactive checkboxes for each item
- Mark items as done
- Count completed and remaining items
- Persist state in `localStorage` (keeps list after refresh)
- Responsive design for mobile and desktop

## Technologies

- Vue 3 (Composition API)
- Vite (fast development server and build)

## Installation

```bash
npm install
npm run dev
# or build
npm run build
```

Open your browser at the address provided by Vite (default http://localhost:5173).

### Project Structure

```
paste-list/
├─ src/
│  ├─ components/
│  │  └─ Checklist.vue
│  ├─ App.vue
│  ├─ main.js
│  └─ style.css
├─ index.html
├─ package.json
└─ vite.config.js
```

### Future Enhancements

Categories and tags for items
Sorting and filtering the list
Export/Import lists (JSON or CSV)
Backend integration (Firebase, Supabase, etc.)
Dark mode toggle
