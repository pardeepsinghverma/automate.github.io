# SmartTank Cloud PWA 💧

Ultra-fast, zero-maintenance, serverless commercial IoT controller for automated overhead water tanks.

## Features
- **0ms Optimistic UI:** Instant tactile feedback with Web Audio API click/chime synthesis.
- **Direct MQTT over WebSockets:** Sub-30ms command latency without intermediary servers.
- **Multi-Device Support:** Switch seamlessly between multiple tanks or borewell motors.
- **QR Code Auto-Pairing:** Scan or enter pairing secret on the hardware box.
- **Offline & Autonomous:** Devices run fully independently even during internet outages.
- **Installable PWA:** Add to Home Screen on iOS & Android for a native app experience.

## Instant 1-Click Deployment to GitHub Pages

1. Create a new repository on GitHub (e.g. `smarttank-app`).
2. Push this folder to your repository:
   ```bash
   git remote add origin https://github.com/<YOUR_USERNAME>/<YOUR_REPO>.git
   git branch -M main
   git push -u origin main
   ```
3. Go to **Settings** → **Pages** on your GitHub repository.
4. Under **Source**, select **Deploy from a branch** → Branch: `main` → Folder: `/ (root)`.
5. Click **Save**. Your app will be live globally at `https://<YOUR_USERNAME>.github.io/<YOUR_REPO>/`!
