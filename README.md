# SilverBullet PDF Export

Exports the current page (or the selection) as a styled PDF through the browser's print dialog.

## Install

In SilverBullet, run `Library: Install` and enter:

```
https://github.com/Tear4Pixelation/silverbullet-pdf-export/blob/main/PLUG.md
```

Then run `Export: PDF`, or pick **PDF** in `Export: Page Or Selection`, and choose **Save as PDF** as the printer. See [PLUG.md](PLUG.md) for details.

## What gets exported
* The page rendered with SilverBullet's own renderer: transclusions and Lua expressions are expanded.
* `> **note**` and `> [!note]` admonitions become callout boxes.
* Code blocks that a plug renders as a widget, such as ` ```excalidraw ` or ` ```canvas `, are rendered by that plug. Widgets run read-only and are printed in light mode.
* The PDF uses a fixed light print style. It does not follow your SilverBullet theme.

The library is a single space-lua page with no build step.

## License
MIT
