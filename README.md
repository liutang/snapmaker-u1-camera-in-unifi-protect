# Snapmaker U1 Camera in UniFi Protect

Add the [Snapmaker U1](https://www.snapmaker.com/)'s built-in camera to UniFi Protect as a continuously recording camera. It gets a status line burned into the video with the time, print progress, layer and finish time.

![The U1 camera in UniFi Protect, with a paused print](images/protect-paused.jpg)

| Printer | Status line |
|---|---|
| Printing | `2026-11-03 01:24:10 PM \| Snapmaker U1 - 42.0% - 40/96 - 1:39 PM` |
| Paused | `2026-11-03 01:31:26 PM \| Snapmaker U1 - Paused - 13.9% - 6/96` |
| Anything else | `2026-11-03 01:45:02 PM \| Snapmaker U1` |

While printing, the line shows progress, current/total layer and the expected finish time. These are the numbers on the printer's own screen. The clock is drawn on every frame, because Protect only draws its own date and name overlay on UniFi cameras.

The printer's own camera view in Fluidd and Mainsail and its timelapses keep working.

## How it works

```
U1 camera feed ──HTTP──> go2rtc (ffmpeg re-encodes and draws the status line) ──ONVIF + RTSP──> UniFi Protect
U1 Moonraker ──> Node-RED, every 10 s ──status.txt──> ffmpeg
```

1. [go2rtc](https://github.com/AlexxIT/go2rtc) runs ffmpeg, which reads the printer's raw H.264 feed over HTTP, draws the status line and re-encodes the video.
2. go2rtc serves the result over RTSP and answers Protect's ONVIF calls, so Protect adopts it as a third-party camera.
3. A Node-RED flow reads the print state from the printer's Moonraker API every 10 seconds and writes the status text to a file. ffmpeg re-reads that file once a second.

## What you need

- **Snapmaker U1 on [Extended Firmware](https://github.com/paxx12-snapmaker-u1/SnapmakerU1-Extended-Firmware).** This guide was tested only on Extended Firmware. It uses the camera feed at `http://PRINTER_IP/webcam/stream.h264` and Moonraker at `http://PRINTER_IP:7125`.
- **A fixed IP address for the printer**, for example a DHCP reservation.
- **A Linux host with Docker and the Docker Compose plugin** for go2rtc, on a network the printer and Protect can both reach. With the overlay, ffmpeg uses about 0.8 of one CPU core on a 4-core Intel CPU from 2013. Without the overlay it uses almost none.
- **UniFi Protect** with third-party camera support.
- **[Node-RED](https://nodered.org/)** that can write to go2rtc's config folder. It's only needed for the status line.
- **The Lato Bold font**, from [Google Fonts](https://fonts.google.com/specimen/Lato). It's free under the SIL Open Font License, and it's close to the look of Protect's own overlay.

Protect records the camera at about 2 Mbps, which is about 22 GB a day.

## Step 1: Check the camera feed

**1. Find the printer's address** on its touchscreen or in your router. This guide calls it `PRINTER_IP`.

**2. Check the feed** from any machine with ffmpeg installed:

```bash
ffprobe -hide_banner -f h264 -i http://PRINTER_IP/webcam/stream.h264
```

It should report `Video: h264 (High)` at `1920x1080`.

**3. Check Moonraker:**

```bash
curl 'http://PRINTER_IP:7125/printer/objects/query?print_stats'
```

It should return JSON with `"state": "standby"` or similar.

## Step 2: Set up go2rtc

**1. Create the config folder** on the Docker host, with a subfolder for the status file:

```bash
mkdir -p go2rtc/config/overlay
cd go2rtc
```

**2. Add the files:**
- `compose.yaml` from this repo's `go2rtc/` folder.
- `go2rtc.yaml` from the same folder, in `config/`. Replace `PRINTER_IP` in both stream lines.
- `Lato-Bold.ttf`, in `config/`.

**3. Set the time zone** for the overlay clock in `compose.yaml`. The `TZ` value is in POSIX form, so it works even if the image has no time zone data. It's set to `UTC0` (UTC); replace it with yours. Examples: `EST5EDT,M3.2.0,M11.1.0` (US Eastern), `CST6CDT,M3.2.0,M11.1.0` (US Central), `PST8PDT,M3.2.0,M11.1.0` (US Pacific), `GMT0BST,M3.5.0/1,M10.5.0` (UK), `CET-1CEST,M3.5.0,M10.5.0/3` (Central Europe).

**4. Set the RTSP password.** Protect uses it to log in. Put it in a `.env` file next to `compose.yaml`:

```bash
echo 'RTSP_PASSWORD=choose-a-long-password' > .env
chmod 600 .env
```

Docker Compose reads `.env` from the same folder when it starts the container. The username is `u1`, set in `go2rtc.yaml`.

**5. Write a first status file.** ffmpeg stops the stream if `status.txt` is missing, so create it before go2rtc starts:

```bash
printf '%s' "%{localtime:'%Y-%m-%d %I:%M:%S %p'} | Snapmaker U1" > config/overlay/status.txt
```

**6. Start go2rtc:**

```bash
docker compose up -d
docker logs go2rtc
```

The log should show `[rtsp] listen addr=:8554` and `[api] listen addr=:1984` and nothing after that.

**7. Check the stream.** Open `http://GO2RTC_HOST:1984` in a browser, where `GO2RTC_HOST` is the Docker host's address, and click **stream** next to `snapmaker_u1`. You should see the camera with the clock in the top left. Or, from a machine with ffmpeg:

```bash
ffprobe -hide_banner -rtsp_transport tcp -i 'rtsp://u1:YOUR_PASSWORD@GO2RTC_HOST:8554/snapmaker_u1'
```

## Step 3: Add it to UniFi Protect

**1. Turn on third-party cameras** in Protect under **Settings → System → Advanced**.

**2. Adopt the camera** with advanced adoption. Use the Docker host's address, port `1984`, username `u1` and the RTSP password.

**3. Set it up.** **Standard** compatibility mode and continuous recording worked in testing. Protect doesn't offer motion detection, PTZ (pan, tilt and zoom) or audio for third-party cameras.

**4. Lock down go2rtc.** In `config/go2rtc.yaml`, uncomment the `allow_paths` line under `api:`, then restart go2rtc:

```bash
docker restart go2rtc
```

That blocks go2rtc's web UI and API, but lets through Protect's ONVIF calls and snapshots. Comment it out again when you need the web UI. Don't set an API password instead: it would block the ONVIF calls too.

## Step 4: Add the status line with Node-RED

**1. Give Node-RED the overlay folder.** Mount go2rtc's `config/overlay` folder in the Node-RED container at `/overlay`. If both run on the same host, add a bind mount to the `node-red` service in Node-RED's `compose.yaml`, then run `docker compose up -d` in that folder to recreate the container:

```yaml
    volumes:
      - /path/to/go2rtc/config/overlay:/overlay
```

If you started Node-RED with `docker run`, add `-v /path/to/go2rtc/config/overlay:/overlay` and recreate it.

If go2rtc's config is on a NAS, mount the NAS folder in Node-RED instead, for example as an NFS volume. Node-RED's official image runs as uid 1000, so that user must be able to write to the folder. On a Synology, give **Everyone** Read & Write on the folder. Without it, writes fail with `EACCES`.

**2. Import the flow.** In Node-RED, open ☰ → **Import**, select `node-red/u1-overlay-flow.json`, and import it into a tab.

**3. Point it at your printer.** Open the **Every 10 s** node and set `msg.moonraker` to `http://PRINTER_IP:7125`. `msg.printer_name` is the name shown on the video.

**4. Deploy.** Within about 10 seconds, `status.txt` holds a line like `%{localtime:'%Y-%m-%d %I:%M:%S %p'} | Snapmaker U1`, and the camera shows the name after the clock.

**5. Start a print to check it.** The layer counter stays at 0 through heating and calibration, then counts up from layer 1. The finish time is the slicer's time estimate for the part not yet printed: now + estimated time × (1 − progress). In testing it was within a minute of the real finish. It moves later by the length of any pause.

The layer count needs the slicer to write `SET_PRINT_STATS_INFO` lines into the G-code. OrcaSlicer and slicers based on it do. Without them, the layer part is left out. Progress needs `M73` lines and the finish time needs the slicer's time estimate in the file.

## Customizing

- **Size and position:** `fontsize`, `x`, `y` and `boxborderw` in the `drawtext` part of the stream line in `go2rtc.yaml`. Restart go2rtc after changing it.
- **Clock format:** the `clock` line at the top of the **Compose overlay text** node. It uses `strftime` codes: for a 24-hour clock use `'%Y-%m-%d %H:%M:%S'`.
- **No overlay:** use the second stream line in `go2rtc.yaml` instead of the first, and restart go2rtc. It copies the video as-is and uses almost no CPU. You can then leave out Node-RED.

## Troubleshooting

- **The live view in Protect shows the top of the image and the rest is green.** The encoder is splitting each frame into slices. Check that `-x264-params sliced-threads=0` is still in the stream line. Recordings and snapshots aren't affected.
- **The stream stops and doesn't come back.** Check that `config/overlay/status.txt` exists: ffmpeg stops if the file is missing when it re-reads it. Recreate it (Step 2.5). The flow always writes to a temp file and renames it into place for this reason. Don't delete `status.txt` while go2rtc is running.
- **The overlay shows `\%` instead of `%`.** In drawtext text a literal `%` must be written as `\%`. The flow already does this; check the file if you write it some other way.
- **The overlay shows only the name during a print.** Node-RED can't reach Moonraker. Check `msg.moonraker` in the **Every 10 s** node, and check Node-RED's debug sidebar for errors.
- **`Overlay write failed: EACCES` in Node-RED's debug sidebar.** Node-RED can't write to the overlay folder. See Step 4.1.
- **The Write overlay file node shows an error about `fs`.** The node loads Node.js's built-in `fs` module through its Setup tab, which needs `functionExternalModules: true` in Node-RED's `settings.js`. That's the default in recent versions.
- **The video's timing is off: it stutters, or plays too fast or too slow.** `-framerate 30` in the stream line must match the feed. Check the feed's rate with the `ffprobe` command in Step 1.
- **The clock is in the wrong time zone.** Check `TZ` in `compose.yaml` (Step 2.3), then recreate the container with `docker compose up -d`.

**To undo:** remove the camera in Protect, then `docker compose down` in the go2rtc folder, and delete the flow in Node-RED. Nothing on the printer changes.

## Files

| Path | What it is |
|---|---|
| `go2rtc/go2rtc.yaml` | go2rtc config: the stream with the overlay, and a commented-out version without it |
| `go2rtc/compose.yaml` | Docker Compose file for go2rtc |
| `node-red/u1-overlay-flow.json` | Node-RED flow that writes the status line |
| `images/` | Screenshot from Protect |

## License

The config files and the Node-RED flow are under the [MIT License](LICENSE). The article and the images are under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/): reuse them freely, with credit and a link back here.
