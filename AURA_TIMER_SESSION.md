# Aura Timer Pro — Claude Code Session Handoff

> Dieses Dokument fasst den aktuellen Entwicklungsstand zusammen und dient als Kontext für die Weiterarbeit in Claude Code.

---

## Projektübersicht

**Aura Timer Pro** ist eine mobile-first Single-Page-App (reines HTML/CSS/JS, kein Framework) für Sport-Timing. Die App läuft vollständig im Browser ohne Build-Step oder Backend.

**Aktuelle Datei:** `aura-timer-pro-v2.html` (eine einzige selbst-enthaltene Datei)

---

## Stack & Abhängigkeiten

| Was | Details |
|-----|---------|
| Framework | Vanilla JS — kein React, kein Bundler |
| Fonts | `Space Mono` (Zahlen/Labels) + `Outfit` (UI-Text) via Google Fonts |
| Storage | `localStorage` — Key `aura_h` für Workout-Verlauf |
| Audio | Web Audio API (`AudioContext`) |
| Screen | Wake Lock API (`navigator.wakeLock`) |
| Animationen | `requestAnimationFrame` Loop, CSS transitions |

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
| **MST** | Mehrstufentest — 6 Stufen automatisch sequenziell (siehe unten) |
| + Eigenes | Modal zur Namensvergabe, derzeit nur Toast-Demo (kein echtes Speichern) |

### MST (Mehrstufentest) — vollständig implementiert
Automatische Sequenz durch 6 Phasen, jede mit eigenem Farbakzent:
1. Aufwärmen — 5:00 — Blau (`--accent-prep`)
2. Grundlage — 5:00 — Grün (`--accent-work`)
3. Schwelle — 4:00 — Gelb (`#facc15`)
4. Intensiv — 3:00 — Orange (`#f97316`)
5. Max — 2:00 — Rot (`--accent-rest`)
6. Auslaufen — 3:00 — Blau (`--accent-prep`)

Gesamtdauer: **22:00**. Outer-Ring zeigt Gesamtfortschritt über alle Stufen.

### UI-Komponenten
- **Doppelter Fortschrittsring**: innerer Ring = Phasenfortschritt, äußerer Ring = Workout-Gesamtfortschritt
- **Aura-Glow**: radialer Hintergrundgradient, Farbe wechselt mit Phase
- **Start-Button**: dynamischer Farbverlauf + Glow-Shadow — synchronisiert mit aktivem Modus/Phase
- **Verlaufspanel**: Slide-up aus Bottom, speichert letzte 20 Sessions in localStorage
- **Toast-System**: kurze Feedback-Messages
- **Paused-State**: Puls-Animation am Ring + ⏸ Overlay

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

**Fonts:**
- Ziffern & Status-Labels: `'Space Mono', monospace`
- Alle anderen UI-Texte: `'Outfit', sans-serif`

**Ring-Geometrie (SVG viewBox 0 0 100 100):**
- Innerer Ring: `r=43.5` → Circumference `273.3`
- Äußerer Ring: `r=48` → Circumference `301.6`
- Strichstärke: `3px` (innen) / `1.2px` (außen)

---

## JS-Architektur

### Globaler State
```js
config = { prep, workMin, workSec, rest, rounds }  // Interval-Konfiguration
cdConfig = { min, sec }                              // Countdown-Konfiguration
mode = 'interval' | 'stopwatch' | 'countdown'
isMSTMode = false | true
isRunning, isPaused, animationFrame
phase = 'prep' | 'work' | 'rest'
currentRound, splits[], lastBeepSec
totalWorkoutDuration, workoutElapsedAtStart
mstStageIndex
workoutHistory[]  // persistiert in localStorage
```

### Wichtige Funktionen
| Funktion | Beschreibung |
|----------|-------------|
| `setMode(m, keepMST?)` | Wechselt Modus, resettet UI |
| `activateMST()` | Aktiviert MST-Ansicht und -State |
| `runMSTStage(idx)` | Startet eine MST-Stufe |
| `runPhase(p, d)` | Startet eine Interval-Phase (prep/work/rest) |
| `updateInterval(ts)` | rAF-Loop für Interval |
| `updateMST(ts)` | rAF-Loop für MST |
| `updateStopwatch(ts)` | rAF-Loop für Stoppuhr |
| `updateCountdown(ts)` | rAF-Loop für Countdown |
| `syncStartBtn(rawHexColor)` | Synchronisiert Start-Button-Farbe/Glow |
| `setAccentColor(cssVar, rawHex)` | Setzt Ring + Aura-Farbe |
| `applyPreset(name, ...)` | Lädt Preset in Wheel-Picker |
| `finish()` | End-Sequenz: Animation, Ton, Vibration, History speichern |
| `playTone(freq, duration)` | Web Audio Beep |

### Wheel-Picker System
- Jedes Wheel ist ein `div.wheel` mit scroll-snap
- Item-Höhe: `40px` — `scrollTop = (value - min) * 40`
- `createWheel(id, min, max, selected)` — generiert DOM + Scroll-Listener
- `setWheelValue(id, val, min)` — setzt Wert programmatisch

---

## Bekannte TODOs / offene Punkte

### Hoch priorisiert
- [ ] **Custom Presets wirklich speichern** — Modal existiert, `saveCustomPreset()` zeigt nur Toast. Implementierung: in `localStorage` speichern + dynamisch als `.preset-btn` in `#presetRow` rendern
- [ ] **Wheel-Picker config-Sync beim Laden** — nach `applyPreset()` synchronisiert die config korrekt, aber beim ersten App-Start wird `config` nicht aus localStorage geladen

### Mittel
- [ ] **Haptic Patterns differenzieren** — `navigator.vibrate()` wird genutzt, aber einheitlich; Work vs Rest vs MST-Stufe könnten unterschiedliche Muster kriegen
- [ ] **MST anpassbar machen** — Stufendauern als editierbare Werte statt hardcoded `MST_STAGES[]`
- [ ] **Verlaufseintrag anklickbar** — History-Items könnten das entsprechende Preset direkt laden
- [ ] **PWA Manifest** — `manifest.json` + Service Worker für echtes "Add to Home Screen" auf Android

### Nice to have
- [ ] **Dark/Light Mode Toggle** — Farbvariablen sind bereits alle in `:root`, wäre ein einfaches Klassen-Toggle
- [ ] **Sound-Auswahl** — Alternativen zu Sinus-Tönen (z.B. Woodblock, Chime) via WaveType oder gesampelte Sounds
- [ ] **Mehrsprachigkeit** — DE/EN Toggle, alle Strings bereits in UI-nahen Funktionen
- [ ] **Shareable URL** — Config als Query-String kodieren für direktes Teilen eines Workouts
- [ ] **Konfetti / Done-Animation** — visuell befriedigendere Finish-Sequenz

---

## Dateistruktur (aktuell)

```
aura-timer-pro-v2.html    ← alles in einer Datei (HTML + CSS + JS)
```

Bei Refaktorierung empfohlen:
```
index.html
css/
  main.css
  components.css
js/
  timer.js        ← rAF-Loops, Audio, WakeLock
  ui.js           ← DOM-Manipulationen, Mode-Switch
  storage.js      ← localStorage-Wrapper
  wheels.js       ← Picker-System
  presets.js      ← Preset-Definitionen + MST_STAGES
```

---

## Entwicklungshinweise für Claude Code

- **Kein Build-Step nötig** — einfach HTML-Datei im Browser öffnen oder `npx serve .` nutzen
- **Testen auf Mobile** — am besten mit Chrome DevTools Device Mode oder echtem Gerät; Wake Lock API nur auf HTTPS oder localhost
- **Audio-Context** — startet erst nach User-Interaction (`audioCtx.resume()` im startWorkout)
- **rAF-Loops** — immer `if(!isRunning || isPaused) return;` am Anfang prüfen
- **ring-Dashoffset** — `0` = Ring voll, `CIRC` = Ring leer (Richtung durch `rotate(-90deg)` auf SVG)

---

*Erstellt in Claude.ai · Aura Timer Pro v2 Session · $(date)*
