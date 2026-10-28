# Isaac Sim on Brev: startup guide

My setup: an NVIDIA Brev instance called `isaac-apples` (1x L40S, 48 GB, Crusoe cloud, VM Mode with Jupyter), running the Isaac Sim 6.2.0 container. I use it only to generate images; training happens on my MacBook.

- Price: about **$1.77/hr while running**, about $0.03/hr for storage while stopped
- Isaac Sim container: `nvcr.io/nvidia/isaac-sim:6.2.0`
- Viewer: `http://<PUBLIC_IP>:8210` (use **http**, not https, and Chrome)
- Repo with the compose file: `~/IsaacSim` on the instance

---

## 1. Start a session

1. Open the Brev console, go to Compute > `isaac-apples`, and click **Start**. Wait until it shows **Running**.
2. Note the **public IP** shown on the instance page. It can change after a stop/start.
3. Scroll to **Cloud Firewall Ports** at the bottom of the page and check that these three rules still exist. Add any that are missing (type the port, click **Expose Port**):

    | Port  | Protocol | What for         |
    | ----- | -------- | ---------------- |
    | 8210  | TCP      | web viewer page  |
    | 49100 | TCP      | stream signaling |
    | 47998 | UDP      | video stream     |

    Tip: set the IP restriction to your own IP if the dropdown allows it. The viewer has no password.

4. Open a terminal on the instance, either way:
    - Brev page: **Open Notebook**, then **Terminal**
    - Mac terminal: `brev open isaac-apples code` (opens VS Code connected to the instance)

## 2. Start Isaac Sim with the streamed viewer

In the instance terminal:

```bash
cd ~/IsaacSim
ISAACSIM_HOST=$(curl -s ifconfig.me) ISAAC_SIM_IMAGE=nvcr.io/nvidia/isaac-sim:6.2.0 \
    docker compose -p isim -f tools/docker/docker-compose.yml up -d --build
```

- The `--build` matters if the public IP changed, because the viewer has the IP built in.
- Isaac Sim needs about **4 minutes** to load. Compose may end with `container isim-isaac-sim-1 is unhealthy`. That is usually just the timeout. Check whether it finished:

```bash
docker logs isim-isaac-sim-1 2>&1 | grep -i "app is loaded"
```

- When you see `Isaac Sim Full Streaming App is loaded.`, start the viewer container if it isn't running:

```bash
docker start isim-web-viewer-1
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

Both `isim-isaac-sim-1` and `isim-web-viewer-1` should be **Up**.

## 3. Open the viewer

In Chrome open `http://<PUBLIC_IP>:8210`. Keep **only one** viewer tab open. If it says "waiting for stream", see Troubleshooting.

## 4. Generate images without the viewer (headless)

For bulk generation you don't need the stream. Stop the streaming stack first so the GPU is free:

```bash
cd ~/IsaacSim
docker compose -p isim -f tools/docker/docker-compose.yml down
```

Then run a script with Isaac Sim's own Python (scripts live in `~/sdg-out`, which is `/workspace` inside the container):

```bash
sudo chown -R 1234:1234 ~/sdg-out

docker run --rm --gpus all --network=host -e ACCEPT_EULA=Y -e PRIVACY_CONSENT=Y \
  --entrypoint /isaac-sim/python.sh \
  -v ~/docker/isaac-sim/cache/main:/isaac-sim/.cache:rw \
  -v ~/docker/isaac-sim/cache/computecache:/isaac-sim/.nv/ComputeCache:rw \
  -v ~/docker/isaac-sim/cache/kit:/isaac-sim/kit/cache:rw \
  -v ~/docker/isaac-sim/logs:/isaac-sim/.nvidia-omniverse/logs:rw \
  -v ~/docker/isaac-sim/config:/isaac-sim/.nvidia-omniverse/config:rw \
  -v ~/docker/isaac-sim/data:/isaac-sim/.local/share/ov/data:rw \
  -v ~/sdg-out:/workspace:rw \
  -u 1234:1234 \
  nvcr.io/nvidia/isaac-sim:6.2.0 /workspace/<your_script>.py
```

Output goes to `~/sdg-out/...` on the instance. (This command is from a test script I haven't confirmed yet. If it errors, fix it before the big run.)

## 5. Get files to / from the instance

- **Upload** (apple-tree model, scripts): drag them into the VS Code explorer window, or use the upload button in the Jupyter file browser.
- **Download** (dataset): zip the folder first (`zip -r dataset.zip ~/sdg-out/<folder>`), then right-click it in the VS Code explorer and choose Download, or download it from the Jupyter file browser.
- Always copy results to your Mac **before** deleting the instance.

## 6. End a session (do this every time)

Click **Stop** at the top right of the instance page in Brev.

- **Stop:** the GPU charge ends. The disk, container image, and caches stay. Only storage (about $0.03/hr) continues.
- **Delete:** everything is gone and charges end. Only do this after the dataset is on my Mac.
- Check the balance in the Brev header. A test run costs a few dollars, but a forgotten running instance costs about $42 per day.

---

## Troubleshooting

| Problem                                             | Likely cause and fix                                                                                                                                                                           |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Compose says `isim-isaac-sim-1 is unhealthy`        | Usually just slow first load. Wait about 4 minutes, check the "app is loaded" log line, then `docker start isim-web-viewer-1`.                                                                 |
| Log shows `Failed to create texture cache`          | Folder permissions. Run `sudo chown -R 1234:1234 ~/docker ~/.cache/ov/hub`, then `docker compose ... down` and `up -d` again.                                                                  |
| Browser says "This site can't be reached" (timeout) | Cloud firewall. Check **Cloud Firewall Ports** in Brev. Make sure the URL starts with **http://**.                                                                                             |
| Viewer page loads but says "Waiting for stream"     | Ports **49100** or **47998 (UDP)** are not open in Brev, or Isaac Sim isn't loaded yet. Also close other viewer tabs.                                                                          |
| Viewer shows an old or wrong IP                     | The public IP changed. Re-run the `up -d --build` command with the new `ISAACSIM_HOST`.                                                                                                        |
| Pull fails with unauthorized or denied              | Log in again: `docker login nvcr.io`, username `$oauthtoken`, password = NGC personal key (create at https://org.ngc.nvidia.com/setup/api-keys, with **NGC Catalog** under Services Included). |
| Something is stuck                                  | `docker logs isim-isaac-sim-1 --tail 40` and `docker logs isim-web-viewer-1 --tail 20`                                                                                                         |

## Useful checks

```bash
nvidia-smi                                   # should show the L40S
docker ps --format 'table {{.Names}}\t{{.Status}}'
sudo ss -ltnup | grep -E '8210|49100|47998'  # what is listening
df -h /                                      # disk space
```

From the Mac (tests that the port is reachable through the firewall):

```bash
nc -vz -w 5 <PUBLIC_IP> 8210
nc -vz -w 5 <PUBLIC_IP> 49100
```

## Security notes

- The viewer and stream have **no password or encryption**. Don't leave the instance running with the ports open to everyone.
- Never paste the NGC key into chats or files. It is shown only once when created.
