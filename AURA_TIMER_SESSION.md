# Aura Timer Pro — Claude Code Session Handoff

> Dieses Dokument fasst den aktuellen Entwicklungsstand zusammen und dient als Kontext für die Weiterarbeit in Claude Code.

---

## Projektübersicht

**Aura Timer Pro** ist eine mobile-first Single-Page-App (reines HTML/CSS/JS, kein Framework) für Sport-Timing. Die App läuft vollständig im Browser ohne Build-Step oder Backend.

**Aktuelle Datei:** `index.html` (eine einzige selbst-enthaltene Datei)
**Deployment:** GitHub → Vercel (auto-deploy bei Push auf `main`)
**Live-URL:** `https://intervaltimer-rt278s-projects.vercel.app`
**Repo:** `https://github.com/RT278/intervaltimer`

---

## Stack & Abhängigkeiten

| Was | Details |
|-----|---------|
| Framework | Vanilla JS — kein React, kein Bundler |
| Fonts | `Outfit` (alle Texte inkl. Zahlen) via Google Fonts — `font-variant-numeric: tabular-nums` für Ziffern |
| QR-Code | `qrcodejs@1.0.0` via jsDelivr CDN |
| Storage | `localStorage` — mehrere Keys (siehe unten) |
| Audio | Web Audio API (`AudioContext`) |
| Screen | Wake Lock API (`navigator.wakeLock`) |
| Animationen | `requestAnimationFrame` Loop, CSS transitions |

### localStorage-Keys
| Key | Inhalt |
|-----|--------|
| `aura_h` | Workout-Verlauf (max 20 Einträge) |
| `aura_cfg` | Interval-Konfiguration `{prep, workMin, workSec, rest, rounds}` |
| `aura_cd` | Countdown-Konfiguration `{min, sec}` |
| `aura_cp` | Custom Presets (Array von `{name, prep, workMin, workSec, rest, rounds}`) |
| `aura_mst` | MST Stufendauern (Array von Sekunden, 6 Werte) |

---

## Aktueller Featurestand

### Modi
- **Interval-Timer** — Prep → Work → Rest Sequenz, konfigurierbar per Scroll-Wheel-Picker
- **Stoppuhr** — mit Lap/Split-Funktion, Best/Worst-Highlighting, Clipboard-Export
- **Countdown** — freie Zeit, Schnellpresets inkl. Pomodoro (25min)

### Presets (Interval)
| Preset | Konfiguration |
|--------|---------------|
| Tabata | 5s Prep · 20s Work · 10s Rest · 8 Runden |
| Kraft  | 5s Prep · 3min Work · 90s Rest · 5 Runden |
| HIIT   | 5s Prep · 40s Work · 20s Rest · 6 Runden |
| **MST** | Mehrstufentest — 6 Stufen automatisch sequenziell |
| + Eigenes | Speichert aktuelle config in localStorage, × zum Löschen |

### MST (Mehrstufentest)
Automatische Sequenz durch 6 Phasen. Stufendauern sind **editierbar** (Tap auf Dauer-Badge → Modal). Gespeichert in `aura_mst`.

| Stufe | Standard | Farbe |
|-------|----------|-------|
| Aufwärmen | 5:00 | Blau |
| Grundlage | 5:00 | Grün |
| Schwelle | 4:00 | Gelb |
| Intensiv | 3:00 | Orange |
| Max | 2:00 | Rot |
| Auslaufen | 3:00 | Blau |

Outer-Ring zeigt Gesamtfortschritt. Gesamtdauer dynamisch berechnet via `getMSTTotal()`.

### UI-Komponenten
- **Doppelter Fortschrittsring**: innerer Ring = Phasenfortschritt, äußerer Ring = Workout-Gesamtfortschritt
- **Aura-Glow**: radialer Hintergrundgradient, Farbe wechselt mit Phase
- **Start-Button**: dynamischer Farbverlauf + Glow-Shadow
- **Verlaufspanel**: Slide-up aus Bottom, speichert letzte 20 Sessions
- **Toast-System**: kurze Feedback-Messages
- **Paused-State**: Puls-Animation am Ring + ⏸ Overlay
- **Share-Modal**: QR-Code der Vercel-URL + URL-Kopierfunktion

---

## CSS Design-Token (`:root`)

```css
--bg: #06080c
--surface: rgba(255,255,255,0.035)
--border: rgba(255,255,255,0.06)
--accent-work: #00e5a0      /* Grün — Work-Phase */
--accent-rest: #ff3d6b      /* Rot — Rest-Phase */
--accent-prep: #4d9fff      /* Blau — Prep / Interval-Modus */
--accent-stopwatch: #fbbf24 /* Gelb — Stoppuhr / MST */
--accent-countdown: #a78bfa /* Lila — Countdown */
--text-main: #f0f4f8
--text-dim: #3d4f61
--text-mid: #7a8fa3
```

**Font:** `'Outfit', sans-serif` — für alle Texte und Zahlen
**Ring-Geometrie (SVG viewBox 0 0 100 100):**
- Innerer Ring: `r=43.5` → Circumference `273.3`
- Äußerer Ring: `r=48` → Circumference `301.6`

---

## JS-Architektur

### Globaler State
```js
config = { prep, workMin, workSec, rest, rounds }  // Interval-Konfiguration
cdConfig = { min, sec }                              // Countdown-Konfiguration
mode = 'interval' | 'stopwatch' | 'countdown'
isMSTMode = false | true
customPresets = []                                   // aus localStorage aura_cp
MST_STAGES = [...]                                   // let, editierbar, Dauern aus aura_mst
isRunning, isPaused, animationFrame
phase = 'prep' | 'work' | 'rest'
currentRound, splits[], lastBeepSec
totalWorkoutDuration, workoutElapsedAtStart
mstStageIndex, mstEditIdx, mstEditMin, mstEditSec
workoutHistory[]
```

### Wichtige Funktionen
| Funktion | Beschreibung |
|----------|-------------|
| `setMode(m, keepMST?)` | Wechselt Modus, resettet UI; Countdown-Wheels via rAF re-synct |
| `activateMST()` | Aktiviert MST-Ansicht und -State |
| `runMSTStage(idx)` | Startet eine MST-Stufe |
| `runPhase(p, d)` | Startet eine Interval-Phase (prep/work/rest) |
| `getMSTTotal()` | Dynamische MST-Gesamtdauer |
| `renderMSTView()` | Rendert `#mstStages` aus MST_STAGES |
| `applyPreset(name, ...)` | Lädt Preset; Wheel-Werte via rAF nach View-Einblendung |
| `renderCustomPresets()` | Rendert User-Presets in `#presetRow` |
| `saveCfg() / saveCdCfg()` | Persistiert config/cdConfig in localStorage |
| `finish()` | End-Sequenz: Animation, Ton, Vibration, History speichern |
| `openShareModal()` | QR-Code-Modal mit App-URL |

### Wheel-Picker System
- Jedes Wheel ist ein `div.wheel` mit `scroll-snap-type: y mandatory`
- Item-Höhe: `32px` — `scrollTop = (value - min) * 32`
- Spacer-Divs (32px) oben und unten für korrekte Zentrierung der Randwerte
- `createWheel(id, min, max, selected)` — setzt `selected`-Klasse direkt bei Erstellung
- `setWheelValue(id, val, min)` — setzt Wert + `selected`-Klasse programmatisch
- `overscroll-behavior: none` — verhindert iOS-Rubber-Band-Effekt
- **Wichtig:** `setWheelValue` und `createWheel` immer aufrufen wenn der Container sichtbar ist, oder in `requestAnimationFrame` wrappen

### Haptic Patterns
- Prep: `[30]`
- Work: `[80, 30, 80]`
- Rest: `[50]`
- MST: stufenweise steigend `[30]` → `[100, 40, 100]` → `[30]`

---

## Deployment

- **Git:** `git add index.html && git commit -m "..." && git push origin main`
- Vercel deployt automatisch bei Push auf `main`
- Git-Config muss `user.email = r_tucholke@gmx.de` haben (sonst Vercel-Fehler: fehlender `githubCommitAuthorLogin`)

---

## Offene TODOs

### Nice to have
- [ ] **Service Worker** — für echtes Offline-Support / PWA-Install auf Android
- [ ] **Verlaufseintrag anklickbar** — History-Items könnten Preset direkt laden
- [ ] **Sound-Auswahl** — Alternativen zu Sinus-Tönen
- [ ] **Shareable URL** — Config als Query-String für direktes Teilen eines Workouts
- [ ] **Konfetti / Done-Animation** — visuell befriedigendere Finish-Sequenz

---

## Entwicklungshinweise

- **Kein Build-Step** — `open index.html` oder `npx serve .`
- **Mobile testen** — Chrome DevTools Device Mode; Wake Lock nur auf HTTPS/localhost
- **Audio-Context** — startet erst nach User-Interaction
- **rAF-Loops** — immer `if(!isRunning || isPaused) return;` am Anfang
- **ring-Dashoffset** — `0` = Ring voll, `CIRC` = Ring leer
- **Wheel-Bug-Muster** — Wheels die im hidden Container initialisiert werden, haben falsche scrollTop. Fix: `requestAnimationFrame(() => setWheelValue(...))` nach `classList.remove('hidden')`

---

*Zuletzt aktualisiert: März 2026*
