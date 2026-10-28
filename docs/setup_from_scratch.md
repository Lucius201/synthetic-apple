# Setting up Isaac Sim on a new Brev instance (from scratch)

Use this if the `isaac-apples` instance was deleted, or you want a fresh one. It follows the exact steps that worked, with the fixes we needed built in.

Time: about 30 minutes. GPU cost for the setup: about $1 to $2 (the L40S ran at $1.77/hr).

---

## 0. Before you start

- [ ] Copy anything you need from the old instance to your Mac (scripts, apple-tree model, datasets). **Deleting an instance deletes its disk.**
- [ ] Brev balance of at least **$5**. Check the number in the Brev header, and add credit under **Billing** if needed.
- [ ] An NVIDIA account that is signed in at brev.nvidia.com.
- [ ] An **NGC personal key** (step 1 below). Without it the container pull fails.

## 1. Create an NGC key (one time, reuse it afterwards)

1. Open https://org.ngc.nvidia.com/setup/api-keys and sign in.
2. Click **+ Generate Personal Key**.
3. Key name: anything (e.g. `isaac-brev`). Expiration: a few months.
4. **Services Included:** select **NGC Catalog**. (If **Private Registry** or **Public API Endpoints** are offered, adding them does no harm.)
5. Click **Generate Personal Key** and **copy the key immediately**. It is shown only once. Keep it in a password manager.
6. Optional: open https://catalog.ngc.nvidia.com/orgs/nvidia/containers/isaac-sim while signed in and accept any terms it asks for.

If you already have a valid key, skip this step.

## 2. Create the instance in Brev

1. Go to https://brev.nvidia.com, **Compute**, **Create Environment** (or use the "Create New Instance" button).
2. In the **Family** list on the left, click **L40S** and keep **GPU count 1x**.
3. Pick a card from the list. The card must show:
    - a **stop/start power icon** (skip cards that say "No stop/start"),
    - **Flexible ports** (not "Fixed ports"),
    - **at least 32 GiB RAM**.

    Last time I used the Crusoe L40S card at about $1.74/hr (147 GB RAM, 8 CPUs). The AWS L40S at about $2.23/hr is the fallback. Prices change, so read the card.

4. On the **Hardware Configuration** screen:
    - **Disk Storage (GiB): 100.** The default is 256, which costs more, and storage is billed even while the instance is stopped. The Isaac Sim container is about 20 GB, so 100 GiB is plenty.
    - **Region:** Any.
    - **Software Configuration:** keep **VM Mode w/ Jupyter**. Don't change it.
    - **Name:** e.g. `isaac-apples`.
5. Check the **Cost Breakdown** (about $1.75/hr total), then click **Deploy**. It may warn that a pre-release option is being used. That's fine.
6. Wait about 7 minutes until the instance shows **Running** and **Built**.

## 3. Open the firewall ports

On the instance page, scroll to **Cloud Firewall Ports** at the bottom. This is the right box. Don't use "TCP/UDP Ports", which only holds the SSH entry.

Add each one: type the port into **Expose Port(s)**, then click **Expose Port**.

| Port  | Protocol | What for         |
| ----- | -------- | ---------------- |
| 8210  | TCP      | web viewer page  |
| 49100 | TCP      | stream signaling |
| 47998 | UDP      | video stream     |

- If the IP dropdown offers your own IP, use it. The viewer has **no password**. "Allow All IPs" works for a quick test, but stop the instance as soon as you finish.
- Check that all three appear in the list under the box.

## 4. Open a terminal on the instance

Either:

- Click **Open Notebook** at the top of the instance page, then open **Terminal** (under "Other"), or
- On your Mac: `brev open <instance-name> code` (needs the Brev CLI, installed with `brew install brevdev/homebrew-brev/brev` and `brev login`; check Brev's docs for the current install steps).

All the commands below run **on the instance**, not on your Mac.

## 5. Check the machine

```bash
nvidia-smi
docker --version
docker compose version
df -h /
```

`nvidia-smi` must show an **L40S**. Docker and Compose should print versions. Brev VMs come with Docker preinstalled. If Docker is missing, install it from NVIDIA's container install docs (Docker plus the NVIDIA Container Toolkit).

## 6. Log in to NGC

```bash
docker login nvcr.io
```

- Username: `$oauthtoken` (type exactly this, with the dollar sign)
- Password: paste your NGC key (nothing shows while you paste)

You should see `Login Succeeded`.

## 7. Get the Docker files and prepare the folders

```bash
cd ~
git clone https://github.com/isaac-sim/IsaacSim
cd IsaacSim

mkdir -p ~/docker/isaac-sim/{cache/main,cache/computecache,cache/kit,config,data,logs,pkg}
mkdir -p ~/.cache/ov/hub
sudo chown -R 1234:1234 ~/docker ~/.cache/ov/hub
sudo chmod -R u+rwX ~/docker ~/.cache/ov/hub
```

The container runs as user **1234**, so these folders must belong to it. Skipping this caused the `Failed to create texture cache` errors last time.

## 8. Start Isaac Sim with the web viewer

```bash
cd ~/IsaacSim
PUBLIC_IP=$(curl -s ifconfig.me)
echo $PUBLIC_IP
ISAACSIM_HOST=$PUBLIC_IP ISAAC_SIM_IMAGE=nvcr.io/nvidia/isaac-sim:6.2.0 \
    docker compose -p isim -f tools/docker/docker-compose.yml up --build -d
```

- The first run downloads the container (about 2.5 minutes last time) and builds the viewer (about 1 minute).
- It will probably end with `dependency failed to start: container isim-isaac-sim-1 is unhealthy`. That is normal on the first start, because compose only waits about 4 minutes and Isaac Sim needs a bit longer. Don't panic.
- Check for the ready line (repeat every minute or two if it's missing):

```bash
docker logs isim-isaac-sim-1 2>&1 | grep -i "app is loaded"
```

You want: `Isaac Sim Full Streaming App is loaded.`

- Check for the old permission error (should print `0`):

```bash
docker logs isim-isaac-sim-1 2>&1 | grep -ci "Failed to create texture cache"
```

If it is not 0, repeat the `chown` and `chmod` from step 7, run `docker compose -p isim -f tools/docker/docker-compose.yml down`, and run the `up` command again.

- Start the viewer container (compose never started it because of the health timeout):

```bash
docker start isim-web-viewer-1
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

Both `isim-isaac-sim-1` and `isim-web-viewer-1` should show **Up**.

## 9. Open the viewer

In **Chrome** on your Mac, open:

```
http://<PUBLIC_IP>:8210
```

- It must be **http://**, not https. Type it in by hand if Chrome changes it.
- Keep **only one** viewer tab open.
- Expected: the Isaac Sim interface with a grid, "RTX - Real-Time 2.0", and about 30 FPS on the L40S.

If the page doesn't load or shows "Waiting for stream", see the troubleshooting section below.

## 10. Stop the instance (every time)

Click **Stop** at the top right of the Brev instance page. GPU billing ends; storage (about $0.012/hr at 100 GiB, $0.03/hr at 256) continues. **Delete** only after your files are on your Mac.

---

## Optional: a headless test render (no viewer needed)

This proves that image generation works. It uses a tiny script (untested, so send me any error). Stop the streaming stack first:

```bash
cd ~/IsaacSim
docker compose -p isim -f tools/docker/docker-compose.yml down
mkdir -p ~/sdg-out
cat > ~/sdg-out/test.py <<'EOF'
from isaacsim import SimulationApp
app = SimulationApp({"headless": True})

import omni.replicator.core as rep

with rep.new_layer():
    rep.create.light(light_type="distant", intensity=3000)
    cam = rep.create.camera(position=(0, -6, 2), look_at=(0, 0, 0))
    rep.create.sphere(position=(0, 0, 0), scale=1, semantics=[("class", "apple")])
    rp = rep.create.render_product(cam, (640, 480))

writer = rep.WriterRegistry.get("BasicWriter")
writer.initialize(output_dir="/workspace/test_out", rgb=True, instance_segmentation=True)
writer.attach([rp])

rep.orchestrator.run_until_complete(num_frames=3)
app.close()
EOF
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
  nvcr.io/nvidia/isaac-sim:6.2.0 /workspace/test.py

ls ~/sdg-out/test_out
```

You should see `rgb_*.png` and instance-segmentation files. Open a PNG in the VS Code explorer to look at it.

---

## Troubleshooting

| Symptom                                                    | Cause and fix                                                                                                                                    |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Compose: `container isim-isaac-sim-1 is unhealthy`         | Usually just the 4-minute timeout. Wait, check the "app is loaded" line, then `docker start isim-web-viewer-1`.                                  |
| `Failed to create texture cache` in the logs               | Folder permissions. Repeat the `chown` and `chmod` (step 7), then `down` and `up -d`.                                                            |
| `unauthorized` or `denied` while pulling                   | Wrong or missing NGC login. Repeat step 6. The key needs **NGC Catalog** in its services. Terms may need accepting on the NGC catalog page.      |
| Browser: "This site can't be reached" / timeout            | Cloud firewall. Check **Cloud Firewall Ports**. The URL must start with **http://**. Also check that the IP restriction matches your current IP. |
| Page loads but "Waiting for stream"                        | Port **49100 (TCP)** or **47998 (UDP)** isn't open, or Isaac Sim isn't loaded yet, or a second viewer tab is open.                               |
| Viewer worked, then the instance was stopped and restarted | The public IP may have changed. Run step 8 again with the new IP (`--build` rebuilds the viewer). Re-check the firewall rules.                   |
| `nvidia-smi` fails                                         | The instance may be on the wrong GPU type. Don't continue. Delete it and create a new one with the L40S.                                         |
| Disk full                                                  | `df -h /`. Clear old data or use a bigger disk on the next deploy.                                                                               |

Handy checks:

```bash
docker logs isim-isaac-sim-1 --tail 40
docker logs isim-web-viewer-1 --tail 20
sudo ss -ltnup | grep -E '8210|49100|47998'
```

From your Mac:

```bash
nc -vz -w 5 <PUBLIC_IP> 8210
nc -vz -w 5 <PUBLIC_IP> 49100
```

## Cost cheat sheet

| Item                       | Cost                                            |
| -------------------------- | ----------------------------------------------- |
| L40S running (Crusoe)      | about $1.74/hr, $1.77/hr with 256 GiB disk      |
| Disk while stopped         | about $0.012/hr (100 GiB) to $0.03/hr (256 GiB) |
| Running 24 h by mistake    | about $42                                       |
| Fresh setup (steps 2 to 9) | about 30 minutes, roughly $1 to $2              |

Rules: stop it when you leave, delete it only after copying your files, and top up Billing before a long run so it can't be cut off in the middle.

## Security notes

- The viewer and the stream have **no password and no encryption**. Don't leave the ports open to everyone for long.
- Never paste the NGC key into chats, screenshots, or files.
