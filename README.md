# BehaveGuard-UPI — Web App (MVP)

Single-file static web app: `index.html`. Firebase Auth (Google + Phone OTP) + Firestore +
three ONNX models running fully in-browser via `onnxruntime-web`.

## 1. Firebase project setup

1. Go to the Firebase Console → **Create a project**.
2. **Build → Authentication → Sign-in method** → enable **Google** and **Phone**.
   - For Phone auth in testing, add your own test phone number under
     Authentication → Sign-in method → Phone numbers for testing (avoids SMS quota issues).
3. **Build → Firestore Database** → Create database (production mode).
4. **Project settings → General → Your apps → Add app → Web (</>)** → copy the config object.
5. Paste that config into `index.html`, replacing the placeholder `firebaseConfig` near the
   top of the `<script type="module">` block:
   ```js
   const firebaseConfig = {
     apiKey: "...", authDomain: "...", projectId: "...",
     storageBucket: "...", messagingSenderId: "...", appId: "..."
   };
   ```
6. **Authentication → Settings → Authorized domains** → add your GitHub Pages domain
   (`yourusername.github.io`) once you know it, or `localhost` for local testing.
7. Deploy the rules in `firestore.rules`:
   ```
   firebase deploy --only firestore:rules
   ```
   (or paste the file's contents into Firestore → Rules in the console).

### Note on the accounts rule
The `mock_banks/{bankId}/accounts/{userId}` rule has been relaxed so any signed-in user can
**write** any account doc (read is still owner-only). This is what makes the full send-money
transaction actually go through end-to-end: completing a payment needs the sender's client to
credit the **receiver's** balance too, and the receiver never opened that particular request, so
an owner-only write rule would silently block that half of the transaction. That's fine here since
it's a mock ledger with no real money — but it does mean any signed-in user could, in principle,
directly edit any account's balance by calling Firestore themselves rather than going through the
app. If you ever want this closer to production-grade, move balance transfers into a Cloud
Function (Admin SDK, bypasses rules) instead of doing the double-write from the client, and put
the owner-only restriction back.

## 2. Add the ONNX models

**You only need the `.onnx` files — the `_config.json` files are optional.** Create a
`models/` folder next to `index.html` and add:
```
models/model_a.onnx
models/model_b.onnx
models/model_c.onnx
```
That's enough to run. Here's what's happening under the hood so you know what you're trusting:

- **Input/output tensor names** are auto-detected directly from each ONNX graph at load time
  (`session.inputNames[0]` / `session.outputNames[0]` via onnxruntime-web) — no JSON needed for this part.
- **Feature order** (which number in the array means what) genuinely can't be read off the graph
  itself, so it's hardcoded in `index.html` to match Part 5 of the build spec exactly
  (search for `FALLBACK_FEATURE_ORDER`). This is the one thing to double-check: if your actual
  trained models expect a *different* column order than the spec, the app will still run — it'll
  just quietly feed the wrong number into the wrong slot, since the shape matches even if the
  meaning doesn't.
- Open the browser console after the app loads — it logs each model's real input/output names
  (`Model A I/O: [...] [...]`) so you can sanity-check them against what you expect.

**If you do have the `_config.json` files** and want the app to read `input_node`/`output_node`/
`feature_order`/`positive_index` from them instead of the hardcoded fallback, add them alongside
the `.onnx` files with the matching names (`model_a_config.json` etc.) and flip this one line near
the top of `index.html`:
```js
const USE_CONFIG_JSON = true; // was false
```

## 3. Run locally

Because of `getUserMedia`/camera and ES module imports, you need to serve over HTTP, not `file://`:
```bash
cd behaveguard
python3 -m http.server 8080
# open http://localhost:8080
```
Camera and geolocation will prompt correctly on `localhost` even without HTTPS.

## 4. Deploy to GitHub Pages

```bash
cd behaveguard
git init
git add .
git commit -m "BehaveGuard-UPI MVP"
git branch -M main
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```
Then: repo **Settings → Pages → Build and deployment → Deploy from a branch → main / (root)**.
Your app will be live at `https://<you>.github.io/<repo>/`.

Add that domain to Firebase Authorized domains (step 1.6 above) or sign-in will fail.

## 5. Debug panel

Hidden behind `#/debug` and gated by `const DEBUG_MODE = false;` near the top of the script.
Flip it to `true` locally to demo fraud scenarios (device override, GPS override, SIM-anomaly
toggle, OTP-mismatch toggle, raw model output viewer). **Set it back to `false` before your
final deploy/push** unless you intentionally want it live during judging.

## 6. What's real vs mock (per the spec)

| Component | Status |
|---|---|
| Google Sign-In + phone OTP | Real — Firebase Auth |
| Device fingerprint + GPS | Real — browser APIs |
| ONNX inference | Real — runs in-browser |
| Fraud scoring (CTX + fusion) | Real |
| Mock banks / balances | Mock — Firestore only |
| Face verification | Real capture, simplified comparison — explicitly not bank-grade KYC |
| Biometric app-lock | Not available in-browser → PIN lock instead |

## 7. Known simplifications worth knowing about before a judged demo

- **Face verification** uses a 24×24 grayscale pixel-intensity vector as a stand-in "descriptor,"
  compared via mean-squared error against a stored baseline. It is *not* real face recognition —
  it's a demo-grade placeholder, clearly labeled as such in the UI and code. A real build would
  swap in a proper in-browser face-embedding library (e.g. a WASM face-landmark model) behind the
  same `Face._descriptorFromVideo()` function.
- **Phone OTP + Google are captured as two separate proofs** rather than cryptographically linked
  via `linkWithCredential` — good enough to gate account activation for a demo, but note if you
  want a single Firebase Auth identity carrying both providers.
- **Model B's UPI-conceptual mappings** (`avs_match`/`cvv_result`/`three_ds_flag` standing in for
  device-known/OTP-verified) are direct 0/1 proxies, exactly as specified — double check these
  align with how each model was actually trained.
