# 🎬 ffmpeg_serviio — FFmpeg wrapper with VAAPI hardware acceleration and caching

[🇬🇧 English](README.md) · [🇷🇺 Русский](README.ru.md)

**A drop-in replacement for `/usr/bin/ffmpeg` for [Serviio Media Server](https://www.serviio.org/)** that adds:

* 🚀 **Hardware acceleration** via VAAPI (Intel Quick Sync / AMD AMF)
* 💾 **Converted file cache** — repeated requests are served instantly
* 🔄 **Cascading fallback** — if GPU transcoding fails, CPU transcoding is attempted
* 📝 **Dual logging** — to a file and to the system log (syslog/journald)
* 🛡️ **Proper handling** of paths containing spaces and `-c:v copy` (remux)

---

## 📋 Features

| Feature                | Description                                                      |
| ---------------------- | ---------------------------------------------------------------- |
| **VAAPI transcoding**  | Uses `/dev/dri/renderD128` for encoding with `h264_vaapi`        |
| **Cascading fallback** | 4 levels: HW+subtitles → HW → SW+subtitles → SW                  |
| **Result caching**     | Converted files are stored in `/fast/cache/converted`            |
| **Stream copy**        | Commands using `-c:v copy` are passed to ffmpeg unchanged        |
| **Syslog integration** | All events are logged via `logger` with the `ffmpeg_serviio` tag |
| **Diagnostics**        | ffmpeg stderr is automatically included in the log               |

---

## ⚙️ Requirements

* **OS:** Linux (tested on Debian/Ubuntu/Proxmox)
* **GPU:** Intel or AMD with VAAPI support
* **Packages:**

  ```bash
  apt install ffmpeg libva-drm2 vainfo acl
  ```
* The **Serviio user** must be a member of the `render` and `video` groups:

  ```bash
  usermod -aG render,video serviio
  ```
* **Device permissions:**

  ```bash
  ls -la /dev/dri/renderD128 /dev/dri/card0
  ```

Check that VAAPI is working:

```bash
vainfo
ffmpeg -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 -i input.mkv -f null -
```

---

## 📦 Installation

### 1. Clone the repository

```bash
git clone https://github.com/rajven/ffmpeg_serviio.git
cd ffmpeg_serviio
```

### 2. Create working directories

```bash
sudo mkdir -p /var/log/ffmpeg /fast/cache/converted
sudo chown serviio:media /var/log/ffmpeg /fast/cache/converted
sudo chmod 755 /var/log/ffmpeg /fast/cache/converted
```

> Replace `serviio:media` with the actual user and group used by your Serviio installation.

### 3. Change start command for serviio

In /opt/serviio/bin/serviio.sh add "-Dffmpeg.location=/usr/local/bin/ffmpeg_serviio" to SERVIIO_JAVA_OPTS

# Setup Serviio specific properties
SERVIIO_JAVA_OPTS="-Djava.net.preferIPv4Stack=true -Djava.awt.headless=true -Dorg.restlet.engine.loggerFacadeClass=org.restlet.ext.slf4j.Slf4jLoggerFacade -Dderby.system.home=$SERVIIO_HOME/library -Dserviio.home=$SERVIIO_HOME -Dffmpeg.location=/usr/local/bin/ffmpeg_serviio -Ddcraw.location=dcraw"


```bash

# Install the wrapper
sudo cp ffmpeg_serviio /usr/local/bin/
sudo chmod 755 /usr/local/bin/ffmpeg_serviio
sudo chown root:root /usr/local/bin/ffmpeg_serviio
```

### 4. Restart Serviio

```bash
sudo systemctl restart serviio
```

---

## 🔧 Serviio Configuration

In the Serviio Console (**Transcoding** → **Video**), make sure that:

* ✅ Transcoding is enabled
* ✅ The correct device profile is selected
* ✅ The transcoding cache path matches `SERVIIO_CACHE_DIR` in the script

The default value in the script is:

```bash
SERVIIO_CACHE_DIR="/fast/cache/serviio/Serviio"
```

---

## 🔄 How It Works

```text
┌─────────────────────────────────────────────────────────┐
│  Serviio calls: ffmpeg [args]                           │
└────────────────────────┬────────────────────────────────┘
                         ▼
              ┌──────────────────────┐
              │ Is this a Serviio    │
              │ request?             │
              │ (path contains       │
              │ SERVIIO_CACHE_DIR)   │
              └──────────┬───────────┘
                    yes /        \ no
                   ▼              ▼
          ┌──────────────┐   ┌──────────────────┐
          │ -c:v copy?   │   │ Run ffmpeg       │
          └──────┬───────┘   │ unchanged        │
            yes /     \ no   └──────────────────┘
               ▼        ▼
     ┌────────────┐  ┌──────────────────┐
     │ Direct     │  │ Check cache      │
     │ remux      │  │ /fast/cache/     │
     └────────────┘  └────────┬─────────┘
                         hit /     \ miss
                           ▼         ▼
                    ┌──────────┐  ┌─────────────────────┐
                    │ Copy     │  │ Cascading fallback: │
                    │ from     │  │ 1. HW + subtitles   │
                    │ cache    │  │ 2. HW               │
                    └──────────┘  │ 3. SW + subtitles   │
                                  │ 4. SW               │
                                  └──────────┬──────────┘
                                             ▼
                                  ┌──────────────────────┐
                                  │ Save to cache        │
                                  │ /fast/cache/converted│
                                  └──────────────────────┘
```

---

## 📝 Logging

The script writes events to **two destinations simultaneously**:

### File log

```text
/var/log/ffmpeg/ffmpeg_serviio.log
```

### System log (syslog/journald)

```bash
# All script entries
journalctl -t ffmpeg_serviio -f

# Errors only
journalctl -t ffmpeg_serviio -p err

# Warnings and above
journalctl -t ffmpeg_serviio -p warning

# Last hour
journalctl -t ffmpeg_serviio --since "1 hour ago"
```

### Log levels

| Level     | When it is used                                                |
| --------- | -------------------------------------------------------------- |
| `info`    | Startup/shutdown, successful operations, cache hits            |
| `warning` | Falling back to another mode, file already exists in the cache |
| `error`   | Insufficient permissions, transcoding failure                  |
| `debug`   | Paths, commands, device groups                                 |

> To enable `debug` logging in rsyslog, add the following to `/etc/rsyslog.conf`:
>
> ```text
> :syslogtag, isequal, "ffmpeg_serviio:" /var/log/ffmpeg_serviio_debug.log
> ```

---

## 📂 Directory Structure

```text
/var/log/ffmpeg/
├── ffmpeg_serviio.log      # Main script log
└── transcoding.log         # List of processed files

/fast/cache/
├── serviio/Serviio/        # Serviio temporary transcoding files
└── converted/              # Long-term cache of converted files
```

---

## 🩺 Troubleshooting

### Serviio does not play the video

```bash
# Check recent errors
journalctl -t ffmpeg_serviio -p err --since "10 min ago"

# Check GPU permissions
ls -la /dev/dri/renderD128
groups $(ps -o user= -p $(pgrep -f serviio))
```

### `exit code: 254`

This usually indicates a path-related problem. Check:

* Permissions on the source file
* Available space on `/fast`
* Correct mounting of network resources

### GPU is not being used

```bash
# Monitor Intel GPU
sudo apt install intel-gpu-tools
sudo intel_gpu_top

# Or via sysfs
cat /sys/class/drm/card0/device/gpu_busy_percent
```

---

## 🛠️ Customization

The main parameters are defined at the beginning of the script:

```bash
LOG_DIR="/var/log/ffmpeg"                         # Log directory
SERVIIO_CACHE_DIR="/fast/cache/serviio/Serviio"  # Serviio temporary cache
CACHE_DIR="/fast/cache/converted"                # Long-term cache
```

Encoding modes are defined in the `build_command` function:

* `hardware_with_subs` / `hardware_no_subs` — VAAPI modes
* `software_with_subs` / `software_no_subs` — libx264 modes

---

## ⚠️ Known Limitations

* The script is designed for **VAAPI** (Intel/AMD). NVIDIA NVENC requires adaptation.
* The cache has no automatic cleanup — configure `cron` or `tmpwatch`.
* With a large number of concurrent transcodings, cache filename collisions are possible (when based on the source filename).

### Example cron job for removing old cache files

```bash
# Remove files older than 30 days
0 3 * * * find /fast/cache/converted -type f -mtime +30 -delete
```

---

## 📄 License

MIT License. Use at your own risk.

---

## 🙏 Acknowledgements

* [Serviio](https://www.serviio.org/) — an excellent media server
* [FFmpeg](https://ffmpeg.org/) — the Swiss Army knife of video processing
* The [Linux VAAPI](https://trac.ffmpeg.org/wiki/Hardware/QuickSync) community for the documentation
