# CWP Classroom Ops ⬡

> **Real-time opening and exit ticket system for the Cyber Warrior Program**  
> Built for adult learners. Optimized for veterans. Hardened by someone who teaches this stuff for a living.

---

## What Is This?

A single-file HTML classroom engagement tool built for the **MyComputerCareer Cyber Warrior Program** (CWP). It replaces the "raise your hand if you got it" method with something that actually tells you who's lost and who's solid — in real time, every night, before you waste 45 minutes reviewing something 90% of the class already knows.

Students open it on their phone or laptop. You watch the dashboard on your shared screen. Questions fly. Points happen. The leaderboard updates live. Someone named `HACKERMAN_99` either earns that callsign or doesn't.

### What It Does

- **Opening Brief** — Retrieval practice at the start of class. Surfaces who retained last night's material before you build on it.
- **Exit Debrief** — Confidence check at end of class. Confirms tonight's core concept landed before you let them go.
- **Live dashboard** — Real-time submission tracking, per-question accuracy bars, and a live leaderboard — all visible on your shared screen without exposing individual answers.
- **Three question types** — Multiple choice, scenario-based, and open reflection. Mix them based on what the topic demands.
- **Speed-bonus scoring** — Correct answers earn 100 pts + up to 50 speed bonus. Fast and right beats just right.
- **Leaderboard with self-rank** — Students see where they stand without seeing each other's answers.

### Who It's For

Adult learners transitioning into cybersecurity. Mostly veterans. People who have operated under pressure and don't need to be coddled — they need to be challenged, respected, and given immediate feedback. This tool is built for that room.

---

## The Tech Stack (Because You'll Be Asked)

| Layer | Choice | Why |
|---|---|---|
| Frontend | Vanilla HTML/CSS/JS | No build tools. No npm. Opens on any device. |
| Realtime backend | Firebase Realtime Database | Free tier, sub-second sync, zero server to manage. |
| Hosting | GitHub Pages | Free. Fast. Already where the code lives. |
| Auth | PIN-gated instructor dashboard | Deterrent-level control. Documented honestly. See `SECURITY.md`. |

Zero recurring costs. Zero dependencies to maintain. One file to edit before each class.

---

## Security Design (Yes, This Was Intentional)

This project was built under SSDLC principles — not because it's a national security application, but because the instructor teaches cybersecurity and the code should model what that looks like in the real world.

Controls applied:

- **XSS prevention** — All Firebase data rendered via `textContent` and `createElement`. Zero `innerHTML` on untrusted strings.
- **No global scope leakage** — Nothing attached to `window`. Firebase config and PIN live in ES module scope only.
- **Input validation** — Callsigns sanitized client-side and enforced server-side via Firebase Rules regex.
- **Question bank validation** — JSON is validated client-side before any Firebase write. Schema, types, and index bounds all checked.
- **Score integrity** — Capped client-side at mathematical maximum. Firebase Rules enforce the same ceiling and block rewrites after submission.
- **Submit lock** — Firebase Rules prevent a student from resubmitting after their score is locked.
- **PIN as deterrent** — The instructor PIN is a plaintext string comparison in module scope. It is not cryptographically strong. This is a documented, conscious tradeoff for a classroom tool with a low-stakes threat model. See `SECURITY.md` for the full residual risk breakdown.

**On Content Security Policy:** A CSP meta tag was implemented in an earlier version and deliberately removed. Firebase Realtime Database uses long-polling as a transport fallback, which dynamically loads scripts from `firebaseio.com` — a domain that cannot be safely added to `script-src` without undermining the policy entirely. GitHub Pages also cannot enforce CSP via HTTP response headers, making a meta tag CSP weaker than a real server-enforced header in any meaningful way. The removal was a conscious operational decision, documented as a residual risk in `SECURITY.md`. The remaining controls are unaffected.

The honest version: if a student reads the source and figures out the PIN, they can see aggregate scores and launch the session early. They cannot see individual answers, modify other students' scores, or do anything that a Firebase Rule doesn't permit. The real security boundary is Firebase — not the PIN.

That's a real security decision, documented like a real security decision. Which is also the lesson.

---

## Setup (Your Own Instance)

This repo contains **no live Firebase credentials**. You wire in your own. Here's how:

### Step 1 — Firebase Project

1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. **Add Project** → name it → disable Google Analytics → Create
3. Left sidebar: **Build → Realtime Database** → Create Database → pick your region → start in test mode

### Step 2 — Apply Security Rules

1. In Realtime Database, click the **Rules** tab
2. Delete the default content
3. Paste in the contents of `firebase-rules.json` from this repo
4. Click **Publish**

### Step 3 — Get Your Config

1. **Project Settings** (gear icon, top left) → scroll to **Your apps**
2. Click the **`</>`** web icon → register an app (name it anything)
3. Copy the `firebaseConfig` object values
4. Open `index.html` and paste each value into `FIREBASE_CONFIG`

### Step 4 — Set Your PIN

Find `INSTRUCTOR_PIN` near the top of the script block. Change the string to whatever you want. Rotate it each cohort.

```js
const INSTRUCTOR_PIN = "YourPinHere";
```

### Step 5 — Host on GitHub Pages

1. Push your edited file to a GitHub repo
2. Repo **Settings → Pages → Source → main branch → / (root)**
3. Your URL appears in ~60 seconds

That URL is the only thing you share with students.

---

## Nightly Workflow

### Before Class (~2 minutes)

1. Open the URL → **INSTRUCTOR** → enter your PIN
2. Click **LOAD QUESTION BANK** on the dashboard
3. Open tonight's `.json` file from your question bank folder → copy all
4. Paste into the left panel → **VALIDATE** → **APPLY TO FIREBASE**
5. Click **LAUNCH SESSION** when students are ready

See `cwp-question-bank-guide.docx` for the full question bank format, Gemini prompt template, and course-by-course writing guidance.

### Running the Session

1. Students open the URL → click **OPERATOR** → enter a callsign → standby screen
2. Hit **LAUNCH SESSION** — all standby screens go live simultaneously
3. Watch the dashboard. Accuracy bars tell you what to cover in debrief.
4. When done: **REVEAL ANSWERS**, debrief, then **RESET SESSION** before the exit ticket

### Switching to Exit Ticket

1. **RESET SESSION** from the dashboard (wipes all scores)
2. Load the exit ticket JSON for tonight via the paste panel
3. Apply → Launch → go again

---

## Question Bank Tips by Course

| Course | Opening Brief Focus | Exit Ticket Focus |
|---|---|---|
| A+ 1201 | Hardware identification, connector types, POST process | Troubleshooting steps, component functions |
| A+ 1202 | OS navigation, command line syntax, networking basics | Configuration steps, error interpretation |
| Network+ | Protocol ports, OSI model layers, subnetting | Packet flow, topology troubleshooting |
| Security+ | CIA triad application, attack types, control categories | Scenario-based threat response |
| CySA+ | CVSS scoring, threat hunting concepts, log analysis | SOC workflow, remediation priority |

Scenario questions hit hardest. Students who can recite facts often freeze when the situation changes. That's the point.

---

## Scoring

| Action | Points |
|---|---|
| Correct MC or Scenario | 100 pts base |
| Speed bonus (correct only) | Up to 50 pts, decays across timer |
| Reflection submitted | 50 pts flat (no right/wrong) |
| Incorrect or timed out | 0 pts |

Maximum per 3-question bank: **350 pts** (2 MC/scenario + 1 reflection)  
Formula: `(MC/scenario count × 150) + (reflection count × 50)`

---

## File Structure

```
cwp-classroom-ops/
├── index.html                   # The entire app — only edit FIREBASE_CONFIG and INSTRUCTOR_PIN
├── firebase-rules.json          # Paste into Firebase Console → Rules
├── SECURITY.md                  # SSDLC documentation and threat model
├── cwp-question-bank-guide.docx # Question bank format reference and Gemini prompt
└── README.md                    # You're reading it
```

---

## Built By

**Shane** — U.S. Army Veteran, Sergeant First Class (Retired)  
Cybersecurity Instructor, MyComputerCareer  
CompTIA A+ | Network+ | Security+ | CySA+ | LPI Linux Essentials | CertNexus CSD | AZ-900 | AI-900  
Working toward: CCNA | CEH | SecAI+

Built with Claude. Secured on purpose. Deployed for the operators who show up every night ready to work.

---

## License

MIT — use it, fork it, teach with it. If you're another CWP instructor and you want the question banks, reach out.

---

*"The leaderboard doesn't lie. Neither does the accuracy bar."*
