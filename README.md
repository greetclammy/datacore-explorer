# Datacore Explorer

Proof of concept [Datacore](https://obsidian.md/plugins?id=datacore) query viewer.

<img width="1156" height="935" alt="Screenshot 2025-10-29 at 06 45 53" src="https://github.com/user-attachments/assets/74aa4f21-50da-4c43-8464-ca9e4fbecc3b" />

## 🖼️ Views

- List
- Card
- Masonry

## 💡 Tips

- `@pages` is implicitly added to each query — need not add it yourself.
- For the viewer to take up the whole viewport width, disable readable line length, or use `max` helper class with a theme that supports Minimal's [helper classes](https://minimal.guide/features/helper-classes).
- File path and tags are horizontally scrollable!
- Press Ctrl/Cmd when hovering over card to bring up page preview.
- Hover over GIF thumbnails to animate them.
- Press on thumbnail to enlarge it (if [Image Toolkit](https://obsidian.md/plugins?id=obsidian-image-toolkit) or a snippet is enabled).
- Search bar filters filenames and #tags (use -#tag to exclude).

## 🗺️ Roadmap

- [ ] Search in file content (support for Obsidian search query syntax, like [Note Gallery](https://github.com/pashashocky/obsidian-note-gallery)).
- [ ] Strip all syntax in text snippet (like [Notebook Navigator](https://github.com/johansan/notebook-navigator/) and [First Line is Title](https://github.com/greetclammy/first-line-is-title)).
- [ ] Add option to skip first line in description (sub-option: only if it's heading).
- [ ] Highlight card on hover (show outline or enlarge it).
- [ ] Change image on hover over thumbnail (for notes with multiple image embeds, not slideshow).
- [ ] Add ability to limit number of results (like in Bases).
- [ ] Add ability to copy links to all results to clipboard (like in Bases).
- [ ] Add table view.
- [ ] Add feed view.
- [ ] Fix setting text fields showing custom outline instead of Obsidian default.
- [x] Show tags/path and text snippet in List view.
