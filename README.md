# Multicast Streaming Scripts

This guide and script sets up a full pipeline using FFmpeg and MediaMTX to restream a multicast MPEG-TS input as:

- RTMP
- RTSP
- HLS
- DASH
- Local `.ts` backup file

---

## 🧰 Requirements

- Ubuntu 20.04+
- Installed: `ffmpeg`, `mediamtx`, `nginx`

---

## ⚙️ MediaMTX Config (`/etc/mediamtx/mediamtx.yml`)

```yaml
rtsp: yes
rtmp: yes
hls:
  enabled: yes
dash:
  enabled: yes

paths:
  your_stream:
    source: publisher
```

Restart MediaMTX:

```bash
killall mediamtx
mediamtx /etc/mediamtx/mediamtx.yml &
```

---

## 📁 Folder Setup

```bash
sudo mkdir -p /var/www/html/hls
sudo mkdir -p /var/www/html/dash
sudo chmod -R 777 /var/www/html
```

---

## 🌐 NGINX Config (Optional for HLS/DASH)

Add to `/etc/nginx/sites-available/default`:

```nginx
server {
    listen 80;
    root /var/www/html;

    location /hls {
        types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
        }
        add_header Cache-Control no-cache;
    }

    location /dash {
        types {
            application/dash+xml mpd;
        }
        add_header Cache-Control no-cache;
    }
}
```

Then restart nginx:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 🚀 Streaming Script (`your_stream-stream.sh`)

```bash
#!/bin/bash

INPUT_URL="udp://@239.17.17.81:1234?localaddr=192.168.212.200&fifo_size=8000000&buffer_size=65536&overrun_nonfatal=1"
PROGRAM_ID="2154"

while true; do
  ffmpeg -re     -i "$INPUT_URL"     -map p:$PROGRAM_ID     -c:v libx264 -preset veryfast -crf 23 -tune zerolatency     -c:a aac -b:a 128k -ac 2     -f tee -map 0:v -map 0:a "[f=flv]rtmp://localhost/live/your_stream|[f=rtsp]rtsp://localhost:8554/your_stream|[f=hls:hls_time=4:hls_list_size=5:hls_flags=delete_segments]/var/www/html/hls/your_stream.m3u8|[f=dash]/var/www/html/dash/your_stream.mpd|[f=mpegts]/var/www/html/your_stream.ts"

  echo "FFmpeg crashed or stopped. Restarting in 3 seconds..."
  sleep 3
done
```

### HLS
```bash
ffmpeg \
  -i "udp://@239.17.17.81:1234?localaddr=192.168.212.200&fifo_size=5000000&overrun_nonfatal=1" \
  -map p:2154 \
  -c:v libx264 -preset veryfast -crf 23 \
  -c:a aac -b:a 128k \
  -f hls \
  -hls_time 4 \
  -hls_list_size 6 \
  -hls_flags delete_segments+program_date_time \
  -method PUT \
  /var/www/html/hls/your_stream.m3u8

```
### DASH
```bash
  ffmpeg \
  -i "udp://@239.17.17.81:1234?localaddr=192.168.212.200&fifo_size=5000000&overrun_nonfatal=1" \
  -map p:2154 \
  -c:v libx264 -preset veryfast -crf 23 \
  -c:a aac -b:a 128k \
  -f dash \
  -window_size 5 \
  -extra_window_size 5 \
  -remove_at_exit 1 \
  -seg_duration 4 \
  /var/www/html/dash/your_stream.mpd
```
### RTMP
```bash
ffmpeg \
  -i "udp://@239.17.17.81:1234?localaddr=192.168.212.200&fifo_size=8000000&buffer_size=65536&overrun_nonfatal=1" \
  -map p:2154 \
  -vf scale=640:360 \
  -c:v libx264 -preset ultrafast -crf 28 -tune zerolatency \
  -c:a aac -b:a 96k \
  -f flv rtmp://localhost/live/your_stream
```
### HTTP
```bash
  ffmpeg \
  -fflags +genpts+igndts \
  -analyzeduration 10000000 \
  -probesize 10000000 \
  -i "udp://@239.17.17.81:1234?localaddr=192.168.212.200&fifo_size=8000000&buffer_size=65536&overrun_nonfatal=1" \
  -map p:2154 \
  -c:v libx264 -preset ultrafast -tune zerolatency -crf 23 \
  -c:a aac -b:a 128k \
  -f mpegts -y /var/www/html/your_stream.ts
```
### RTSP
```bash
ffmpeg \
  -fflags +genpts+igndts \
  -analyzeduration 10000000 \
  -probesize 10000000 \
  -i "udp://@239.17.17.81:1234?localaddr=192.168.212.200&fifo_size=8000000&buffer_size=65536&overrun_nonfatal=1" \
  -map p:2154 \
  -c:v libx264 -preset veryfast -crf 23 -tune zerolatency \
  -c:a aac -b:a 128k \
  -f rtsp rtsp://localhost:8554/your_stream
```

Make it executable:

```bash
chmod +x your_stream-stream.sh
./your_stream-stream.sh
```

---

## 🔗 Playback URLs

- HLS: `http://<your-ip>/hls/your_stream.m3u8`
- DASH: `http://<your-ip>/dash/your_stream.mpd`
- RTSP: `rtsp://<your-ip>:8554/your_stream`
- RTMP: `rtmp://<your-ip>/live/your_stream`
- Local TS: `/var/www/html/your_stream.ts`
