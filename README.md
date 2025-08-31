# Rummikid (React Native, iOS-first)

## 1. Disclaimer
1. **Not affiliated with Rummikub® or its trademark owners.**  
2. This project exists purely as a **personal home project** to experiment with AI technologies.  
3. The app is **not public and will not be released publicly** on the App Store or Play Store.  

## 2. Objective
Rummikid is a **kid-friendly mobile app** that helps children playing Rummikub determine whether they meet the **initial 30-point meld requirement**.

My primary customer target is my son Leo, aged 5 at the time of building this app, who is a massive fan of Rummikub but does not know how to count properly and wishes he did not have to expose his game to other players in other to know whether he has his initial 30 points or not.

The app uses the iPhone camera in landscape mode to scan tiles, upload a photo to a backend, and provide a **clear thumbs-up/thumbs-down result**.

The app also includes a **Cheat Mode toggle**. When activated, the backend **ignores existing patterns** and instead analyzes all visible tiles to find the **best possible combination(s)** that reach or exceed 30 points. If it’s not possible, the app shows the **maximum achievable total** with suggested melds.

Backend: **API Gateway → Lambda (TypeScript) → AWS Bedrock**  
Client: **React Native (iOS only for v1)**, Android support considered later.

## 3. In/Out of Scope (v1)

**In scope**
- iOS app (React Native, landscape mode only).
- Camera preview with auto-capture every ~2s when stable.
- Normal Mode: evaluates formed melds in the photo.
- Cheat Mode: recombines all tiles into best meld set.
- Single backend API handling both modes.
- No persistent storage of images.

**Out of scope**
- Android release (kept in mind for portability).
- On-device AI/ML.
- Accounts, push notifications, or analytics.

## 4. User Stories

- *As a child,* I want to point the camera at my tiles and see a big 👍 if I have 30 points.  
- *As a parent,* I want to enable **Cheat Mode** so my child can learn which combinations work.  
- *As a player,* I want to see what’s possible even if I don’t have 30 yet.  

## 5. UX Summary

### 5.1. Home Screen

Brand “Rummikid”, mascot, Start Checking button.

![Home Screen](docs/designs/Screen%201%20Home.png)

### 5.2. Scan Screen

Live camera with wide guide box, brand top-left, Cheat Mode toggle top-right, status light centered at top.

![Scan Screen](docs/designs/Screen%202%20Scan.png)

### 5.3. Loading Screen**

Brand centered, bouncing tile animation, “Checking your points…”.

![Loading Screen](docs/designs/Screen%203%20Loading.png)

### 5.4. Result Screen (Normal)

**Success** → 👍, “Yes! You have 30 points!”, Try Again button.

![Success Screen (Normal)](docs/designs/Screen%204%20Success%20normal%20mode.png)

**Fail** → 👎, “Not yet — try again!”, Try Again button.

![Fail Screen (Normal)](docs/designs/Screen%205%20Fail%20normal%20mode.png)

### 5.5. Result Screen (Cheat Mode)

**Success** → 👍, headline “Try these tiles for X points!”, multiple meld rows shown with green outline.

![Success Screen (Cheat)](docs/designs/Screen%206%20Success%20cheat%20mode.png)

**Fail** → 👎, headline “Only X points possible right now”, best meld rows shown with **neutral outline**.

![Fail Screen (Cheat)](docs/designs/Screen%207%20Fail%20cheat%20mode.png)

## 6. Functional Requirements

### Modes
- **Normal Mode**:  
  - Evaluates only formed melds.  
  - Returns total + meets30 boolean.  

- **Cheat Mode**:  
  - Ignores visible layout, treats all tiles as a rack.  
  - Computes best non-overlapping meld set maximizing total points.  
  - If ≥30 → returns meld rows highlighted in green.  
  - If <30 → returns best achievable rows with neutral outline.

### Capture
- Auto-capture at most every 2s when:  
  - brightness acceptable,  
  - sharpness above threshold,  
  - low motion for ~2s,  
  - enough tile-like shapes in ROI.  
- Pause capture while a request is in flight.

### Privacy
- No images stored server-side.  
- No third-party tracking.  

## 7. Non-Functional Requirements

- **Latency**: ≤2s end-to-end.  
- **Availability**: 99% is fine for hobby use.  
- **Battery**: lightweight on-device checks only.  
- **Accessibility**: large type, thumbs icons, minimal text.

## 8. Architecture

**Client (React Native, iOS landscape only)**
- Camera: `react-native-vision-camera`.  
- Networking: multipart upload via fetch/axios.  
- State machine: idle → capture → uploading → result.

**Backend**
- API Gateway `POST /analyze`.  
- Lambda (TypeScript, Node 20):  
  - Parse multipart request.  
  - Validate image (≤1.5 MB JPEG).  
  - Call **Bedrock multimodal model** for tile detection/OCR.  
  - Normalize tiles into `{number, color, joker}`.  
  - Run meld evaluator (normal) or solver (cheat).  
  - Return JSON with melds + totals.

## 9. API Contract

**Endpoint:** `POST /analyze`

**Request (multipart/form-data):**
- `image`: JPEG/PNG, ≤1.5 MB.  
- `payload` (JSON string):  
  - `mode`: `"normal"` or `"cheat"`  
  - `clientMeta`: `{ appVersion, deviceModel }` (optional)

**Response — Normal Mode (success example):**
```json
{
  "mode": "normal",
  "meets30": true,
  "totalPoints": 33,
  "melds": [
    {
      "id": "m1",
      "type": "run",
      "color": "blue",
      "numbers": [10, 11, 12],
      "points": 33,
      "tileIds": ["t1", "t2", "t3"]
    }
  ],
  "tiles": [
    { "id": "t1", "number": 10, "color": "blue", "joker": false },
    { "id": "t2", "number": 11, "color": "blue", "joker": false },
    { "id": "t3", "number": 12, "color": "blue", "joker": false }
  ]
}
```

**Response — Cheat Mode (≥30 possible):**
```json
{
  "mode": "cheat",
  "meets30": true,
  "bestCombination": {
    "totalPoints": 41,
    "melds": [
      { "id": "m1", "type": "run", "color": "orange", "numbers": [5,6,7], "points": 18, "tileIds": ["t4","t5","t6"] },
      { "id": "m2", "type": "set", "number": 9, "colors": ["red","blue","black"], "points": 23, "tileIds": ["t7","t8","t9"] }
    ]
  }
}

```

**Response — Cheat Mode (<30 only):**
```json
{
  "mode": "cheat",
  "meets30": false,
  "bestCombination": {
    "totalPoints": 26,
    "melds": [
      { "id": "m1", "type": "run", "color": "blue", "numbers": [3,4,5], "points": 12, "tileIds": ["t1","t2","t3"] },
      { "id": "m2", "type": "set", "number": 8, "colors": ["red","orange"], "points": 14, "tileIds": ["t10","t11"] }
    ]
  }
}
```

**Errors:**
```json
{ "error": { "code": "BAD_IMAGE", "message": "Blurry/too dark. Try again." } }
{ "error": { "code": "NO_TILES", "message": "No tiles detected." } }
```

## 10. Data & Rules

**Tile model**
- number: 1–13 or null (joker).
- color: red, blue, black, orange.
- joker: boolean.

**Meld rules**
- Run: same color, consecutive numbers, ≥3 tiles.
- Set: same number, all colors unique, 3–4 tiles.
- Joker substitutes missing tile.
- Score = sum of numbers (joker adopts value).

**Cheat solver**
- Generate candidate runs/sets with jokers flexibly assigned.
- Choose non-overlapping melds maximizing total points.
- Tie-break: fewer melds with higher total.

## 11. Acceptance Criteria

**Normal Mode:**
- Clear formed melds detected correctly.
- Correct meets30 boolean + total.

**Cheat Mode:**
- ≥30 found → returns meld rows with green highlight.
- <30 found → returns max total + neutral outlined meld rows.

**Performance:**
- Median latency ≤2s.
- Works with typical table lighting.

## 12. Risks & Mitigations

- **Detection accuracy** → mitigate with guide box, stability gating, retries.
- **Latency from Bedrock** → mitigate with compressed images, fast fail if >10s.
- **UI complexity** → keep kid-simple (thumbs + headline + rows).

## 13. Android Considerations

- RN stack is cross-platform; libraries chosen for iOS will work on Android.
- Keep layout + API consistent for easy Android build later.
- Frame processors may need native Kotlin counterpart.
