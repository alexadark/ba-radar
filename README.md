# BA Radar

Personal PWA listing the week's events in Buenos Aires, found by a weekly scan of Instagram accounts.

- App: static `index.html`, served by GitHub Pages. Installable on desktop and phone.
- Data: a separate private repository, read and written through the GitHub API with a fine-grained token stored on each device.
- Scan: a routine kept in the private data repository, run weekly by a T3 Code scheduled task on the Mac. Claude orchestrates and judges; a Codex agent collects posts with computer use in Chrome, with the logged-in Instagram session.

Release: bump `APP_VERSION` in index.html and version.json together so installed copies show the update banner.
