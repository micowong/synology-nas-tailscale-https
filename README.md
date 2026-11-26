# Securing Synology NAS Access with a Single Valid TLS Certificate (Internal LAN + Tailscale VPN)

Modern home labs and self-hosted setups often face an awkward dilemma: exposing a Synology NAS to the public internet invites automated scanning and rapid exploitation, while locking it down behind a VPN often breaks valid HTTPS certificates or forces awkward split configurations.

This guide shows how to keep your Synology NAS **off the public internet** (no port forwarding required) while using **one valid TLS certificate** (`<yourname>.synology.me`) seamlessly across both your **internal LAN** and **external devices connected via Tailscale**.
<img width="1177" height="823" alt="Frontpage" src="https://github.com/user-attachments/assets/674172bc-334c-4025-b475-a90635c12da2" />

---

## 1. Why This Matters: Rapid Exploitation & the 2026 PHP Chaos

Exposing web services, management interfaces, or PHP-backed applications directly to the public internet has become substantially riskier. Automated tooling and AI-assisted exploit development compress the window between vulnerability disclosure and mass scanning/exploitation from weeks or days to hours.

In late 2026, core PHP vulnerabilities underscored how quickly critical flaws cascade across web stacks and appliances:
* **CVE-2026-109618 & CVE-2026-109619 ("PHP2Shell"):** Disclosed in October 2026 across PHP 5.4 through 8.5.11 (patched in 8.2.35, 8.3.36, 8.4.27, and 8.5.12), involving partial initialization during out-of-memory handling and stale request state reuse across worker processes—leading to severe impact including remote code execution on exposed services.
* **Follow-on PHP & Web Flaws:** Demonstrated how quickly follow-up bugs and exploit chains emerge once attackers focus on a runtime or web stack.

**The takeaway for a Synology NAS:** Even if your NAS isn't running a vulnerable PHP app today, leaving DSM, Reverse Proxy, or web apps exposed via router port forwarding creates an unnecessary attack surface. Using a private overlay VPN (Tailscale) for remote access—combined with internal DNS and a valid certificate—gives you encrypted, zero-port-forwarding access without sacrificing browser trust.

---

## 2. Architecture Overview

1. **Certificate Authority (DSM DDNS + Let's Encrypt):** DSM manages `<yourname>.synology.me` and renews its Let's Encrypt certificate automatically via `DNS-01` validation without needing any inbound router ports forwarded to your NAS.
2. **Internal LAN Resolution:** Your router points LAN clients' DNS to **Synology DNS Server**, which resolves `<yourname>.synology.me` to the NAS's private LAN IP (`192.168.x.y`).
3. **External Access via Tailscale:** Your NAS runs Tailscale as a `/32` subnet router for `192.168.x.y`, and Tailscale's restricted nameserver setting directs queries for `<yourname>.synology.me` to `192.168.x.y` over the VPN tunnel.
4. **Same Certificate Everywhere:** Because both internal and external clients connect to `<yourname>.synology.me` (resolving to `192.168.x.y`), the exact same valid certificate is presented in both locations.
<img width="1042" height="507" alt="Synology_Tailscale_Architecture-v5 drawio" src="https://github.com/user-attachments/assets/184ff34b-8982-4090-8ef4-e7b16677fbed" />

---

## 3. Configuration Steps

### Part A: Internal Network & Certificate Setup

#### 1. Configure DSM DDNS and Certificate
1. In DSM, go to **Control Panel > External Access > DDNS** and click **Add**.
   <img width="2076" height="1500" alt="Control Panel-External Access-DDNS" src="https://github.com/user-attachments/assets/be922d11-eabf-41cc-b923-c972079a44ea" />
2. Choose **Synology** as the Service Provider, enter your desired hostname (`<yourname>.synology.me`), and check **Get a certificate from Let's Encrypt and set it as default** (or generate/assign one under **Control Panel > Security > Certificate**).
3. **Zero-port-forwarding renewal:** Unlike third-party custom domains (which require inbound port 80 forwarded for HTTP-01 validation), **Synology DDNS supports `DNS-01` validation**. DSM automatically obtains and renews your `<yourname>.synology.me` certificate every 90 days over outbound connections—no open inbound ports required.

#### 2. Install & Configure Synology DNS Server
1. Open **Package Center**, search for **DNS Server**, and install it.
   <img width="1755" height="682" alt="DNS Server" src="https://github.com/user-attachments/assets/11af8400-fa40-4b21-86da-2f0371afb94e" />
2. Launch **DNS Server** and ensure the DNS service is enabled.
3. Go to **Zones** and create a **Primary zone** (**Forward zone**) for `<yourname>.synology.me` and enter `192.168.x.y` (your Synology NAS) as the **Primary DNS server**.
   <img width="1701" height="1673" alt="DNS Server-new Zone" src="https://github.com/user-attachments/assets/640bd5c2-a1fb-4b67-9970-7aa9a8a637a8" />
4. Then select the zone, click **Edit > Resource record > Create**, choose **A**, leave **Name** blank (so it targets `<yourname>.synology.me` itself). Disable **Limit zone transfer** and **Limit zone update** unless you specifically need them.
   <img width="1376" height="768" alt="DNS Server-Create A record" src="https://github.com/user-attachments/assets/9b7e7557-fef7-43f1-9825-21111fbc451d" />
5. Go to **Resolution**, enable resolution service, configure **Enable forwarders** (such as `1.1.1.1` and `8.8.8.8`), and if you enable **Limit source IP service**, make sure to include both your local LAN subnet (e.g., `192.168.1.0/24` or your chosen LAN subnet) and Tailscale's CGNAT range (`100.64.0.0/10`)—otherwise external Tailscale devices won't be able to query your NAS for DNS.
    <img width="1974" height="1738" alt="DNS Server-Resolution" src="https://github.com/user-attachments/assets/32e2628d-50d3-476c-820f-350d1703d2ad" />


#### 3. Redirect Router DHCP / Internal DNS to Synology
1. Log in to your **router's admin panel**.
2. In your **LAN / DHCP settings**, set the primary **DNS Server** handed out to internal clients to `192.168.x.y` (your Synology NAS).
3. **Avoid common subnet collisions:** If your home LAN uses a very common subnet like `192.168.1.0/24`, consider switching your router's LAN subnet to a less common range (such as `192.168.50.x` or `10.10.20.x`). Because this guide routes `192.168.x.y/32` (or your NAS's LAN IP) over Tailscale, an uncommon subnet reduces the chance of IP conflicts when connecting from external Wi-Fi networks (hotels, cafes, offices, or friends' networks). *(If you change your NAS's LAN IP, update your Synology DNS A record and Tailscale `/32` route to match.)*
4. Reconnect your test device (or renew its DHCP lease) so it picks up the new DNS server and subnet settings.

#### 4. Test Internal DNS & HTTPS Access
1. On a device connected to your internal LAN, open `https://<yourname>.synology.me:5001` (or your service URL) in a **web browser** and confirm it loads cleanly with a valid certificate.
2. For a quick double-check that DNS itself isn't cached or falling back elsewhere, run:

```bash
nslookup <yourname>.synology.me
```

**Expected result:** The browser loads without certificate warnings, and `nslookup` resolves `<yourname>.synology.me` to `192.168.x.y`.

---

### Part B: External Access via Tailscale VPN

#### 1. Create a Tailscale Account
1. Sign up at [tailscale.com](https://tailscale.com) (free tier available for personal use).
2. Note your tailnet settings in the **Tailscale Admin Console**.

#### 2. Install Tailscale on Synology & Fix Initial Reauthentication
1. Open **Package Center** on DSM and check for **Tailscale**. If it is not listed in your region/DSM build (or you prefer the latest official build), download the `.spk` package matching your NAS CPU architecture from Tailscale's official package server and choose **Manual Install** in Package Center.
   <img width="1757" height="1156" alt="TailScale NAS" src="https://github.com/user-attachments/assets/ba478c6e-fc7a-490f-873c-ce5fcfd3c6c1" />
2. Check the **Tailscale package UI** in DSM (and your **Tailscale Admin Console > Machines** list) first to see if the NAS is already connected.
   <img width="1406" height="795" alt="TailScale NAS-Running" src="https://github.com/user-attachments/assets/0f888253-1e60-47a1-a101-87e752169b94" />
4. If the Tailscale UI shows *"Your device's key has expired. Reauthenticate this device..."* and the **Reauthenticate** button does nothing:
   - In DSM, go to **Control Panel > Terminal & SNMP** and temporarily enable **SSH**.
   - SSH into your NAS and authenticate via CLI:

```bash
ssh <your_dsm_admin_user>@192.168.x.y
sudo tailscale up
```

   - Open the authentication URL printed in the terminal in a browser and sign in to your Tailscale account.
   - Verify connectivity in the Tailscale UI/Admin Console (or via `tailscale status` in SSH), then go back to **Control Panel > Terminal & SNMP** and **disable SSH** once finished.

#### 3. Advertise & Approve the `/32` Subnet Route for the NAS
To route only to the Synology NAS's LAN IP (`192.168.x.y/32`) rather than your entire home subnet:

1. Temporarily enable **SSH** if disabled, connect to the NAS, and advertise the `/32` route using `tailscale set` (which preserves existing flags rather than resetting them to defaults; use `tailscale up --advertise-routes=192.168.x.y/32` if your version or state requires `up`):
   <img width="2033" height="879" alt="Control Panel-SSH" src="https://github.com/user-attachments/assets/35d01aa4-a7bd-4330-94a3-cc139466694b" />
   <img width="688" height="384" alt="SSH-tailscale advertise-routes" src="https://github.com/user-attachments/assets/5e83e044-9853-4f35-9746-a0ce2155173b" />
   
```bash
sudo tailscale set --advertise-routes=192.168.x.y/32
```

   - Disable **SSH** again afterward if you don't need it left on.
2. Go to the **Tailscale Admin Console > Machines**, select your Synology NAS, open **Edit route settings**, and **approve** `192.168.x.y/32`.
   <img width="1376" height="768" alt="TailScale-Approve Subnets" src="https://github.com/user-attachments/assets/c1650d00-13ed-4652-8c37-913f7065ad3a" />
   
3. Check **Control Panel > Security > Firewall** in DSM: if the firewall is enabled, allow traffic from Tailscale's CGNAT range (`100.64.0.0/10`) and your LAN subnet.

#### 4. Configure Tailscale Nameservers for `<yourname>.synology.me`
1. In the **Tailscale Admin Console**, go to **DNS > Nameservers**.
2. Add a **Restricted nameserver** (split DNS) for domain `<yourname>.synology.me` pointing to `192.168.x.y`.
   <img width="1376" height="768" alt="TailScale-DNS-Nameservers" src="https://github.com/user-attachments/assets/05d3d48b-9d25-49ee-b9ca-afaa327e4f3a" />

#### 5. Install Tailscale on Mobile & Test Both Networks
1. Install the **Tailscale** app on iOS or Android and sign in to the same tailnet.
2. Ensure **Subnet routes** are accepted (enabled by default on iOS and Android).
3. **Test 1 — Internal Wi-Fi:** Connect your phone to home Wi-Fi and open `https://<yourname>.synology.me:5001` (or your service URL). Confirm it loads with a valid certificate.

   <img width="344" height="768" alt="Android-Wifi access NAS" src="https://github.com/user-attachments/assets/fa2b6d97-7403-40eb-834d-58c36c01e7c2" />
   
4. **Test 2 — External Cellular (VPN Active):** Turn off Wi-Fi on your phone so it uses mobile data, ensure **Tailscale VPN is connected**, and open `https://<yourname>.synology.me:5001`. Confirm it loads over the VPN with the **exact same valid certificate**.

   <img width="344" height="768" alt="Android-Cellular access NAS" src="https://github.com/user-attachments/assets/27ddf95d-3ba9-466d-95f9-1f45212aa33b" />


---

## 4. Other Notes & Alternatives

### Alternative 1: Use Tailscale `100.x.y.z` IP Instead of a `/32` Subnet Route

Instead of advertising `192.168.x.y/32` as a subnet router, devices on the tailnet can connect directly to the Synology NAS's Tailscale IP (`100.x.y.z`).

#### Pros
* **Zero `192.168.x.y` subnet collisions:** If you connect from hotel, cafe, office, or friend Wi-Fi that also uses `192.168.x.0/24`, you avoid local IP/subnet conflicts.
* **No subnet router approval overhead:** You don't need to advertise or approve a `/32` LAN route.

#### Cons
* **Requires Split-Horizon / Views in DNS:** Internal LAN clients need `<yourname>.synology.me` to resolve to `192.168.x.y`, while VPN clients need it to resolve to `100.x.y.z`—requiring DNS **Views** (split-horizon DNS) or separate DNS rules.
* **Watch for post-upgrade / TUN / firewall hiccups:** After DSM or Tailscale updates, verify Tailscale is running and DSM firewall rules allow `100.64.0.0/10`; note that DSM's `TUN` configuration task is specifically needed when other Synology packages on the NAS initiate *outgoing* connections to other `100.x.y.z` nodes, though checking service/network/TUN state after major DSM upgrades is good practice.

### Alternative 2: Tailscale Serve (`.ts.net` HTTPS)
* Tailscale Serve can provide HTTPS for a tailnet `.ts.net` hostname (and DSM can pull/renew a `.ts.net` certificate via `tailscale configure synology-cert` in a root Task Scheduler job, or you can import/manage certificates manually).
* **Trade-off:** It works well for tailnet-only access, but adds extra certificate/task setup and renewal maintenance if you prefer keeping `<yourname>.synology.me` unified across LAN and VPN.

### Alternative 3: DSM Reverse Proxy with Subdomains (`<app>.<yourname>.synology.me`)
* You can route multiple apps or containers to clean subdomains like `photos.<yourname>.synology.me` via DSM's Reverse Proxy.
* **Trade-off:** Rather than issuing a separate certificate per subdomain, you can use a **wildcard certificate** (`*.<yourname>.synology.me`) on your Synology DDNS certificate—however, adding reverse proxy rules, subdomain DNS entries/views, and per-service headers (such as WebSockets) increases overall setup and troubleshooting complexity.
