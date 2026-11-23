# media/

- `app.png` — the Electron frontend
- `results.png` — query results view
- `demo-web.mp4` — *not here yet*

The original `demo.mp4` is a 198-second, 1478x1080 screen recording at ~2.9 Mbps
(73 MB), which is above GitHub's 50 MB warning threshold and would sit in git
history permanently. Compress it first:

    ffmpeg -i demo.mp4 -vf "scale=1280:-2,fps=15" \
           -c:v libx264 -crf 28 -preset slow -pix_fmt yuv420p \
           -c:a aac -b:a 96k -movflags +faststart media/demo-web.mp4

Under about 10 MB, GitHub plays it inline in the README. The `.gitignore`
already allows `media/demo-web.mp4` through while blocking other `.mp4` files.
