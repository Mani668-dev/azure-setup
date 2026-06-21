# 📸 Screenshots Folder

## Purpose

This folder will contain all Azure Portal screenshots for the documentation.

## Folder Structure

```
screenshots/
├── README.md (this file)
├── 01-azure-ad/          (15 screenshots)
├── 02-acr/               (9 screenshots)
├── 03-aks/               (12 screenshots)
├── 04-jump-server/       (9 screenshots)
└── 05-blob-storage/      (13 screenshots)
```

## How to Add Screenshots

1. **Follow the Screenshot Guide**: See `docs/SCREENSHOT-GUIDE.md`
2. **Take screenshots** as you follow the setup documentation
3. **Save with provided filenames** (e.g., `screenshot-01-portal-home.png`)
4. **Organize into folders** by documentation section
5. **Commit and push** to this repository

## Screenshot Requirements

- **Format**: PNG
- **Resolution**: 1920x1080 or higher
- **Size**: < 500KB per image (compress if needed)
- **Sensitive Data**: Blur subscription IDs, tenant IDs, emails

## Quick Start

```bash
# After taking screenshots, add them
git add screenshots/
git commit -m "Add Azure Portal screenshots"
git push origin main
```

## Tools for Taking Screenshots

### Windows
- **Win + Shift + S**: Snipping Tool
- **Win + Print Screen**: Full screen

### Mac
- **Cmd + Shift + 4**: Selection
- **Cmd + Shift + 3**: Full screen

### Linux
- **Ksnip** or **Flameshot**

## Status

- [ ] 01-azure-ad/ - 0/15 screenshots
- [ ] 02-acr/ - 0/9 screenshots
- [ ] 03-aks/ - 0/12 screenshots
- [ ] 04-jump-server/ - 0/9 screenshots
- [ ] 05-blob-storage/ - 0/13 screenshots

**Total**: 0/58 screenshots needed

---

*For detailed instructions, see [SCREENSHOT-GUIDE.md](../docs/SCREENSHOT-GUIDE.md)*
