# WebRTC_ROS2_ws
ROS 2 workspace containing WebRTC Python server and recorder node for streaming audio and video over WebRTC and publishing to ROS 2 topics.

## Overview
This workspace bridges WebRTC streams into ROS 2, enabling remote audio/video capture and playback. It consists of audio processing packages (audio_common family) and WebRTC integration packages.

**Target Platform:** ROS 2 Jazzy Jalisco on Ubuntu 24.04

## Installation
```bash
git clone https://github.com/NeefMartin/WebRTC_ROS2_ws.git
cd WebRTC_ROS2_ws
git submodule update --init --recursive
rosdep install --from-paths src --ignore-src -r -y
colcon build
source install/setup.bash
```

## Packages

### 1. audio_common_msgs
**Purpose:** Defines ROS 2 message types for audio data.

**Messages:**
- `AudioData` — Raw audio sample data
- `AudioDataStamped` — Audio data with timestamp and frame_id
- `AudioInfo` — Audio metadata (sample rate, codec, channels, etc.)

**Usage:** Imported as a dependency by other audio packages.

---

### 2. audio_capture
**Purpose:** Captures audio from microphone or file using GStreamer and publishes as ROS 2 messages.

**Node:** `audio_capture_node`

**Publishes:**
- `/audio/audio` (`audio_common_msgs/AudioData`) — Raw audio data
- `/audio/info` (`audio_common_msgs/AudioInfo`) — Audio metadata

**Parameters (via launch files):**
- Audio device/source (e.g., default input device or file path)
- Output format and sample rate

**Launch Files:**
```bash
# Capture from default microphone to topic
ros2 launch audio_capture capture.launch.py

# Capture from file
ros2 launch audio_capture capture_to_file.launch.py

# Capture and encode to WAV
ros2 launch audio_capture capture_wave.launch.py
```

**Dependencies:** GStreamer libraries (`libgstreamer1.0-dev`, `libgstreamer-plugins-base1.0-dev`)

---

### 3. audio_play
**Purpose:** Subscribes to audio topics and outputs to speakers using GStreamer.

**Node:** `audio_play_node`

**Subscribes:**
- `/audio/audio` (`audio_common_msgs/AudioData`) — Audio data to play
- `/audio/info` (`audio_common_msgs/AudioInfo`) — Audio format info

**Launch Files:**
```bash
# Play audio from topic to default speaker
ros2 launch audio_play play.launch.py
```

**Dependencies:** GStreamer libraries, ALSA support

---

### 4. sound_play
**Purpose:** High-level audio playback node providing Python/C++ API for sounds, speech synthesis, and file playback.

**Node:** `soundplay_node.py`

**Subscribes:**
- `/robotsound` (`sound_play_msgs/SoundRequest`) — Sound commands

**Services:**
- Speech synthesis, OGG/WAV playback, built-in sound triggers

**Features:**
- Built-in beep/chirp sounds
- File playback (OGG, WAV)
- Speech synthesis via Festival
- Python and C++ client bindings

**Launch Files:**
```bash
ros2 launch sound_play soundplay_node.launch.xml
```

**Usage Example (Python):**
```python
from sound_play.libsoundplay import SoundClient

soundclient = SoundClient()
soundclient.playWave("/path/to/file.wav")
soundclient.say("Hello World")
soundclient.play(rosgraph.rospack.RosPack().get_path('sound_play') + '/sounds/beeep.ogg')
```

**Dependencies:** Festival (speech synthesis), GStreamer, Python 3 with Gi bindings

---

### 5. robot_stream
**Purpose:** WebRTC signaling server and HTTPS file server. Relays WebRTC signaling between browsers and the recorder node.

**Components:**
- **Python Server** (`server.py`) — Handles:
  - HTTPS serving of web pages (streamer.html, viewer.html)
  - WebRTC signaling relay (offer/answer/ICE candidates)
  - Viewer registration and fan-out connections

**Ports:**
- **8080 (HTTPS)** — Web UI and file serving
- **8765 (WSS)** — WebRTC signaling

**Features:**
- Auto-generates self-signed TLS certificates on first run
- Supports multiple viewers with dedicated peer connections
- JSON DataChannel for arbitrary ROS command passing

**Launch:**
```bash
# Default ports (8080 HTTPS, 8765 WSS)
ros2 launch robot_stream robot_stream.launch.py

# Custom ports
ros2 launch robot_stream robot_stream.launch.py ws_port:=8765 http_port:=8080
```

**Certificate Generation:**
- Auto-generates on first run if missing
- Uses `openssl` if available, falls back to Python `cryptography` package
- Includes SAN entries for localhost and detected LAN IP

**Web Pages:**
- `streamer.html` — Browser client that captures camera/microphone and sends WebRTC offer
- `viewer.html` — Browser viewer that displays the remote stream
- `index.html` — Home page with links

---

### 6. webrtc_recorder
**Purpose:** C++ node that receives WebRTC media tracks and DataChannel messages, publishing them as ROS 2 topics.

**Node:** `webrtc_recorder_node`

**Publishes:**
- `/camera/image/raw` (`sensor_msgs/Image`) — Video frames (BGR8 decoded)
- `/audio/raw` (`audio_common_msgs/AudioData`) — Audio data (Opus passthrough)
- `/ros_control` (`std_msgs/String`) — JSON DataChannel messages

**Architecture:**
- Registers as a "streamer" client with `robot_stream` server
- Receives WebRTC offers and establishes ICE connections
- Decodes H.264 video → OpenCV Mat → ROS Image message
- Passes through Opus audio frames directly
- Forwards DataChannel strings as ROS messages

**Dependencies:**
- ROS 2 development libraries
- libwebrtc (WebRTC native library)
- OpenCV (`cv_bridge`)
- GStreamer (optional, for audio handling)

**Launch:**
Typically launched as part of the `robot_stream.launch.py` file:
```bash
ros2 launch robot_stream robot_stream.launch.py
```

---

## Quick Start Guide

### 1. **Stream Setup (Complete System)**
Stream audio and video from a browser to ROS 2 topics:

```bash
source install/setup.bash

# Start the complete system (server + recorder node)
ros2 launch robot_stream robot_stream.launch.py

# In another terminal, start recording
ros2 bag record /camera/image/raw /audio/raw /ros_control
```

Then open a browser:
- **Streamer (local):** `https://localhost:8080/streamer.html`
- **Viewer:** `https://localhost:8080/viewer.html` or pass the VM IP address

### 2. **Audio Capture Only**
```bash
source install/setup.bash
ros2 launch audio_capture capture.launch.py
```

Listen to captured audio:
```bash
ros2 launch audio_play play.launch.py
```

### 3. **Sound Playback**
```bash
source install/setup.bash
ros2 launch sound_play soundplay_node.launch.xml

# In Python script or CLI:
ros2 topic pub /robotsound sound_play_msgs/SoundRequest '{sound: 0}'
```

### 4. **Record Streamed Media**
```bash
# Terminal 1: Start streaming system
ros2 launch robot_stream robot_stream.launch.py

# Terminal 2: Record to bag
ros2 bag record /camera/image/raw /audio/raw /ros_control

# Terminal 3: View topics
ros2 topic list
ros2 topic echo /camera/image/raw
```

---

## Dependencies & System Requirements

### System Dependencies
Install via `rosdep`:
```bash
rosdep install --from-paths src --ignore-src -r -y
```

### Key Dependencies
- **GStreamer:** Audio/video codec support
  - `libgstreamer1.0-dev`
  - `libgstreamer-plugins-base1.0-dev`
  - `gstreamer1.0-plugins-{base,good,ugly}`

- **WebRTC:** Native WebRTC libraries (for webrtc_recorder)
  - `libwebrtc-dev` (or built from source)

- **OpenCV:** Image processing
  - `libopencv-dev`
  - `cv-bridge` ROS package

- **Festival:** Speech synthesis (optional, for sound_play)
  ```bash
  sudo apt-get install festival
  ```

- **Python 3:** For robot_stream and sound_play
  - `python3`
  - `python3-websockets`
  - `python3-gi` (GObject introspection for audio)

---

## Topic Reference

| Topic | Message Type | Publisher | Description |
|-------|--------------|-----------|-------------|
| `/camera/image/raw` | `sensor_msgs/Image` | webrtc_recorder_node | Decoded H.264 video frames |
| `/audio/raw` | `audio_common_msgs/AudioData` | webrtc_recorder_node | Opus audio data |
| `/ros_control` | `std_msgs/String` | webrtc_recorder_node | JSON DataChannel messages |
| `/audio/audio` | `audio_common_msgs/AudioData` | audio_capture_node | Captured audio |
| `/audio/info` | `audio_common_msgs/AudioInfo` | audio_capture_node | Audio metadata |
| `/robotsound` | `sound_play_msgs/SoundRequest` | (external) | Commands to soundplay_node |

---

## Build & Test

### Build All Packages
```bash
colcon build
```

### Build Specific Package
```bash
colcon build --packages-select webrtc_recorder
```

### Run Tests
```bash
colcon test
colcon test-result --verbose
```

---

## Troubleshooting

### GStreamer Errors
- Ensure GStreamer plugins are installed: `sudo apt-get install gstreamer1.0-plugins-{base,good,ugly,bad}`
- Check audio device: `arecord -l` (alsa-utils)

### WebRTC Certificate Issues
- Certificates are auto-generated and cached in `src/robot_stream/`
- Delete `cert.pem` and `key.pem` to force regeneration

### WebRTC Connection Failures
- Ensure firewall allows ports 8080 and 8765
- Check that both machines can reach each other on the network
- Verify `/etc/hosts` or DNS resolution for the VM hostname

### Audio Not Playing
- Check ALSA configuration and device list: `aplay -l`
- Verify PulseAudio or ALSA is running properly

---

## License
Packages are licensed under BSD (audio_common) and MIT (robot_stream, webrtc_recorder).

