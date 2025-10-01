**Important Notes:**
- Run commands sequentially unless otherwise specified (e.g., different terminals).
- Many commands require `sudo` privileges.
- **Adapt IP addresses** (`192.168.70.135`, `192.168.70.129`, `10.0.0.2`, `10.200.200.1`, `10.200.200.2`), **file paths** (OAI/FlexRIC locations, media files like `test.mp4` or `reference_440hz.wav`), **network interface names** (`oaitun_ue1`, host interfaces like `ens33`, Docker bridge `oai-cn5g`), and the **UE IMSI** (`001010000000001`) to match your specific environment.

---

## Phase 1: Common OAI/FlexRIC Component Initialization

```bash
# Terminal 1: Start OAI 5G Core Network
cd ~/oai-cn5g
sudo docker compose up -d

# Optional: Monitor AMF logs (Ctrl+C to exit)
# sudo docker logs oai-amf -f

# Terminal 2: Start FlexRIC Near-RT RIC
cd ~/
./flexric/build/examples/ric/nearRT-RIC

# Terminal 3: Start OAI gNB Simulator
cd ~/oai/cmake_targets/ran_build/build # Adjust path if needed
sudo ./nr-softmodem RFSIMULATOR=server \
    -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf \
    --gNBs.[0].min_rxtxtime 6 --rfsim --sa

# Terminal 4: Start OAI UE Simulator
cd ~/oai/cmake_targets/ran_build/build # Adjust path if needed
sudo RFSIMULATOR=127.0.0.1 ./nr-uesoftmodem \
    -r 106 --numerology 1 --band 78 -C 3619200000 \
    --nokrnmod --rfsim --sa --uicc0.imsi 001010000000001

# --- Wait for UE to connect (Monitor AMF logs in Terminal 1) ---
```

---

## Phase 2: Common Media Streaming Network Setup

This setup uses network namespaces and virtual Ethernet pairs to bridge traffic between the UE simulator and the host network where the media server runs.

```bash
# Host Terminal: Install Prerequisites (if not already done)
# sudo apt update && sudo apt upgrade -y
# sudo apt install ffmpeg iproute2 net-tools wget nano jq -y

# Host Terminal: Download and Prepare MediaMTX
cd ~
rm -f mediamtx mediamtx.yml mediamtx_*.tar.gz # Clean previous versions
wget https://github.com/bluenviron/mediamtx/releases/latest/download/mediamtx_linux_amd64.tar.gz
tar xvf mediamtx_linux_amd64.tar.gz
chmod +x mediamtx

# Host Terminal: Configure MediaMTX
nano ~/mediamtx.yml
# Ensure 'rtspAddress' is set to an IP reachable from the UE namespace via veth/NAT
# Example: rtspAddress: 192.168.70.129:8554
# Save and exit (Ctrl+O, Enter, Ctrl+X)

# Host Terminal (New): Start MediaMTX Server
cd ~
./mediamtx
# (Leave this running)

# Host Terminal: Set Up UE Network Namespace
sudo ip netns add ue_ns
sudo ip link set oaitun_ue1 down
sudo ip link set oaitun_ue1 netns ue_ns

# Host Terminal: Configure Interface Inside Namespace
sudo ip netns exec ue_ns bash -c '
    ip link set lo up
    ip link set oaitun_ue1 up
    ip addr add 10.0.0.2/24 dev oaitun_ue1
'

# Host Terminal: Create Virtual Ethernet Pair
sudo ip link add veth_host type veth peer name veth_ue
sudo ip link set veth_ue netns ue_ns
sudo ip addr add 10.200.200.1/24 dev veth_host
sudo ip link set veth_host up

# Host Terminal: Configure veth Inside Namespace and Set Default Route via Host
sudo ip netns exec ue_ns bash -c '
    ip addr add 10.200.200.2/24 dev veth_ue
    ip link set veth_ue up
    # Route traffic destined outside the namespace via the host bridge IP
    ip route replace default via 10.200.200.1 dev veth_ue
'

# Host Terminal: Enable IP Forwarding and Configure NAT
sudo sysctl -w net.ipv4.ip_forward=1
# Find Docker bridge name (e.g., br-xxxx) if not 'oai-cn5g'
OAI_BRIDGE_INTERFACE=$(docker network inspect oai-cn5g | jq -r '.[0].Options."com.docker.network.bridge.name"')
# Or set manually: OAI_BRIDGE_INTERFACE="br-xxxxxxxxx"
sudo iptables -A FORWARD -i veth_host -o $OAI_BRIDGE_INTERFACE -j ACCEPT
sudo iptables -A FORWARD -i $OAI_BRIDGE_INTERFACE -o veth_host -j ACCEPT
sudo iptables -t nat -A POSTROUTING -s 10.200.200.0/24 -o $OAI_BRIDGE_INTERFACE -j MASQUERADE
```

---

## Phase 3: Media Streaming Execution (Choose Video OR Audio)

Perform the steps in **EITHER** 3.A **OR** 3.B.

### **3.A: VIDEO Streaming**

```bash
# Host Terminal (New): Start Streaming VIDEO Source with FFmpeg
# Replace test.mp4 with your video file
ffmpeg -re -stream_loop -1 -i test.mp4 -c copy -f rtsp rtsp://192.168.70.129:8554/stream
# (Leave this running)

# Host Terminal: Receive VIDEO Stream Inside UE Namespace
# Enter the namespace
sudo ip netns exec ue_ns bash

# --- Inside UE Namespace Shell (Video) ---
# Verify connectivity to MediaMTX server
# ping -c 4 192.168.70.129

# Receive Video Example (saves 10s timestamped):
ffmpeg -rtsp_transport tcp -i rtsp://192.168.70.129:8554/stream -c copy -t 10 "ue_video_$(date +%Y%m%d_%H%M%S).mp4"

# (Stop with Ctrl+C)

# Exit the namespace shell
exit
# --- End of Inside UE Namespace Shell (Video) ---
```

### **3.B: AUDIO Streaming**

```bash
# Host Terminal (New): Start Streaming AUDIO Source with FFmpeg
# Replace reference_440hz.wav with your audio file
ffmpeg -re -stream_loop -1 -i reference_440hz.wav -c:a aac -b:a 128k -f rtsp rtsp://192.168.70.129:8554/audio_stream
# (Leave this running)

# Host Terminal: Receive AUDIO Stream Inside UE Namespace
# Enter the namespace
sudo ip netns exec ue_ns bash

# --- Inside UE Namespace Shell (Audio) ---
# Verify connectivity to MediaMTX server
# ping -c 4 192.168.70.129

# Receive Audio Example (saves 30s with specific parameters):
ffmpeg -rtsp_transport tcp \
    -i rtsp://192.168.70.129:8554/audio_stream \
    -copyts \
    -max_delay 500000 \
    -c:a pcm_s16le \
    -af aresample=async=1000,asetpts=PTS-STARTPTS \
    -t 30 \
    received_440hz.wav

# (Stop with Ctrl+C)

# Exit the namespace shell
exit
# --- End of Inside UE Namespace Shell (Audio) ---
```

---

## Phase 4: Common Cleanup

```bash
# Stop MediaMTX (Ctrl+C in its terminal)
# Stop FFmpeg streaming source (Ctrl+C or 'q' in its terminal)
# Stop OAI/FlexRIC components (Ctrl+C in Terminals 2, 3, 4)

# Remove iptables rules (use correct bridge interface name)
sudo iptables -D FORWARD -i veth_host -o $OAI_BRIDGE_INTERFACE -j ACCEPT
sudo iptables -D FORWARD -i $OAI_BRIDGE_INTERFACE -o veth_host -j ACCEPT
sudo iptables -t nat -D POSTROUTING -s 10.200.200.0/24 -o $OAI_BRIDGE_INTERFACE -j MASQUERADE

# Delete the Virtual Ethernet Pair
sudo ip link delete veth_host

# Delete the UE Namespace (ensure no processes are running inside it)
sudo ip netns delete ue_ns

# Stop OAI 5G Core
# cd ~/oai-cn5g
# sudo docker compose down
```
