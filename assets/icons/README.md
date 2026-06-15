# VMem SVG Icon Set

All icons designed as scalable vector graphics for the VMem RAM Disk project.

## Application Icons

| File | Usage | Size |
|------|-------|------|
| `vmem-app.svg` | Main application icon (taskbar, window) | 256x256 |
| `installer.svg` | Installer wizard icon | 256x256 |

## System Tray Icons

| File | Usage | Size |
|------|-------|------|
| `vmem-tray.svg` | Default tray icon | 16x16 |
| `vmem-tray-connected.svg` | Tray: service connected (green dot) | 16x16 |
| `vmem-tray-disconnected.svg` | Tray: service disconnected (red dot) | 16x16 |

## Navigation Icons

| File | Usage | Size |
|------|-------|------|
| `nav-dashboard.svg` | Left nav: Dashboard/Home | 24x24 |
| `nav-performance.svg` | Left nav: Performance monitor | 24x24 |
| `nav-settings.svg` | Left nav: Settings | 24x24 |
| `nav-about.svg` | Left nav: About | 24x24 |

## Action Icons

| File | Usage | Size |
|------|-------|------|
| `action-create.svg` | Create new RAM disk (+ button) | 24x24 |
| `action-open-folder.svg` | Open disk in Explorer | 24x24 |
| `action-eject.svg` | Unmount/eject disk | 24x24 |
| `action-delete.svg` | Delete/trash | 24x24 |
| `action-refresh.svg` | Refresh/retry connection | 24x24 |
| `action-save.svg` | Save settings | 24x24 |

## Status Icons

| File | Usage | Size |
|------|-------|------|
| `status-connected.svg` | Service connected indicator | 16x16 |
| `status-disconnected.svg` | Service disconnected indicator | 16x16 |
| `status-warning.svg` | Warning triangle (memory alert) | 24x24 |

## Preset Icons

| File | Usage | Size |
|------|-------|------|
| `preset-browser.svg` | Browser cache preset | 24x24 |
| `preset-temp.svg` | System TEMP preset | 24x24 |
| `preset-dev.svg` | Dev compile cache preset | 24x24 |

## Disk & Feature Icons

| File | Usage | Size |
|------|-------|------|
| `disk-drive.svg` | Generic disk card icon | 24x24 |
| `disk-mounted.svg` | Mounted disk with green check | 24x24 |
| `memory-chip.svg` | Memory usage display | 24x24 |
| `speed-gauge.svg` | I/O speed indicator | 24x24 |
| `snapshot.svg` | Snapshot/backup feature | 24x24 |
| `service.svg` | Windows Service status | 24x24 |
| `theme-auto.svg` | Theme auto-detect | 24x24 |

## Tool Icons

| File | Usage | Size |
|------|-------|------|
| `tool-log.svg` | Open log directory | 24x24 |
| `tool-repair.svg` | Repair WinFsp driver | 24x24 |
| `tool-export.svg` | Export diagnostic report | 24x24 |

## Design Guidelines

- **Style**: Stroke-based (2px), matching Fluent Design / Segoe Fluent Icons
- **Colors**: Navigation/action icons use `currentColor` for theme adaptability
- **App icon**: Uses brand gradient (#6366F1 Indigo → #8B5CF6 Violet)
- **Status dots**: Green (#22C55E), Red (#EF4444), Yellow (#F59E0B)
- **Accent**: Cyan (#22D3EE) for speed/performance indicators

## ICO Conversion (Build Time)

For Windows .ico files (tray/taskbar), convert at build time:

```bash
# Using Inkscape CLI or svg2ico
inkscape vmem-app.svg -w 256 -h 256 -o vmem-app-256.png
inkscape vmem-app.svg -w 48 -h 48 -o vmem-app-48.png
inkscape vmem-app.svg -w 32 -h 32 -o vmem-app-32.png
inkscape vmem-app.svg -w 16 -h 16 -o vmem-app-16.png
# Then combine into .ico using ImageMagick
magick vmem-app-{16,32,48,256}.png vmem-app.ico
```
