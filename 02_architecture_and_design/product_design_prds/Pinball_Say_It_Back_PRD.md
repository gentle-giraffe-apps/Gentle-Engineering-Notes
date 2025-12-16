# Pinball Say-It-Back — Initial PRD

## 1. Overview

**Pinball Say-It-Back** is an educational iOS game that combines a gentle pinball-style physics experience with active language recall.  
Players interact with a soft 3D pinball board by **tilting their device** and optionally **touching interactive elements** on the board.

The core learning loop pauses gameplay at frequent intervals to present a **flashcard-style recall moment**, prompting the player to *say the vocabulary word out loud*.  
Speech is evaluated using **Apple’s Speech-to-Text APIs**, with feedback presented via a **confidence meter** rather than binary success/failure.

The game is designed to be calm, forgiving, and accessible—especially for young learners.

<img
  src="images/PinballSayItBackPRD.png"
  alt="Pinball Say It Back PRD"
  width="1024"
/>

<img
  src="images/PinballSayItBack.png"
  alt="Pinball Say It Back UI"
  width="1024"
/>

---

## 2. Target Audience

- Children ages **3–7**
- Early language learners
- Parents and educators looking for low-friction, voice-based learning
- Future extension: bilingual learners

---

## 3. Core Learning Loop

1. **Ball contacts a vocabulary object**
   - Example objects: apple, banana, pear, mango
   - Object visually highlights
   - **Prerecorded audio** plays once (e.g., “Apple”)

2. **Physics play phase**
   - Ball continues moving through the board
   - Player tilts the device to influence gravity
   - Optional touch interactions (bumpers, paddles)

3. **Ball capture**
   - Ball falls into one of many board holes / capture sockets
   - Game **pauses immediately**
   - All background audio and motion stop

4. **Recall phase**
   - Flashcard appears:
     - Large centered image of the object
     - No text label
   - UI overlay text: **“Say it”**
   - Microphone icon gently pulses
   - Game remains silent

5. **Speech evaluation**
   - Apple Speech-to-Text listens for a short window
   - Recognized word compared loosely to target word
   - Confidence meter fills based on similarity

6. **Feedback + continuation**
   - Correct / high confidence:
     - Player gains a **Save Token**
     - Subtle positive animation
   - Incorrect / low confidence:
     - Save Token removed (if any)
   - Gameplay resumes with physics restored

---

## 4. Game Mechanics

### 4.1 Lives / Save Tokens

- Player starts with **2 Save Tokens**
- Save Tokens:
  - Represent automatic ball saves
  - Visually shown as glowing shields or bumpers
- When the ball would fall out:
  - The game may randomly intervene **only if Save Tokens exist**
  - Intervention is visually obvious (slow motion, glowing flipper)

### 4.2 Physics Interaction

#### Device Tilting
- Uses accelerometer + gyroscope
- Tilting the device:
  - Tilts the entire pinball board
  - Gravity direction changes smoothly
- Tilt limits prevent extreme angles
- Motion is slow, floaty, and forgiving

#### Touch Interaction (Optional)
- On-screen elements:
  - Large bumpers
  - Soft paddles
  - Gentle blockers
- Touching elements:
  - Activates a brief bounce or deflection
  - Never requires fast reflexes
- Touch is optional; tilt alone is sufficient

---

## 5. Board Design (Visual Description)

### Overall Style
- Soft, toy-like 3D aesthetic
- Rounded edges, pastel colors
- Calm lighting, no harsh contrasts

### Board Layout
- Slightly angled rectangular board
- Multiple **clearly visible holes** distributed evenly
- Holes act as intentional capture points
- No “instant loss” feeling

### Vocabulary Objects
- Large, friendly 3D icons:
  - Apple (red, shiny)
  - Banana (yellow, curved)
  - Pear (green)
  - Mango (orange)
- Objects subtly animate when hit
- Objects glow briefly when audio plays

### Capture Holes
- Circular with soft glow
- Visual depth cue (dark center)
- When ball enters:
  - Gentle suction animation
  - Soft “click” sound (then silence)

---

## 6. Flashcard UI (Recall State)

### Screen State
- Board blurred or dimmed in background
- Centered card with:
  - Large image only
  - No text label
- Overlay text:
  - “Say it” (rounded font)
- Microphone icon:
  - Pulsing ring animation

### Confidence Meter
- Horizontal or circular meter
- Fills gradually as speech is processed
- Color transitions:
  - Red → Yellow → Green
- No harsh failure messaging

---

## 7. Audio Design

### Prerecorded Audio
- Clear, friendly human voice
- One-word recordings only
- Played only once per object hit

### Speech Recognition
- Uses Apple Speech-to-Text
- Loose matching rules:
  - Phoneme similarity
  - Partial matches allowed
- No penalties for silence or mic errors

---

## 8. Accessibility & UX Principles

- No timers during recall
- No negative sounds
- Visual feedback always paired with motion
- Calm pacing throughout
- Designed to reduce frustration

---

## 9. MVP Scope

- 1 board
- 4 vocabulary words
- Tilt interaction
- Capture-based recall
- Confidence meter
- Save Tokens mechanic

Out of scope for MVP:
- Scoring
- Levels
- Multiplayer
- Ads

---

## 10. Future Extensions (Not MVP)

- Bilingual modes
- Custom word packs
- Parent dashboard
- Additional boards
- Animal sounds / verbs

---

## 11. Summary

Pinball Say-It-Back uses **embodied play + active recall** to transform flashcards into a physical, joyful learning experience.  
The design emphasizes forgiveness, clarity, and confidence—allowing children to focus on speaking and learning without pressure.

## One-Screen Visual Prompt for AI Image / Prototype Generation

---

## Primary Visual Prompt (Copy-Paste Ready)

**Prompt:**

A calm, child-friendly **3D educational pinball game screen** for iOS, shown in portrait orientation.

The background is a **soft, toy-like pinball board** with rounded edges, pastel colors, and gentle lighting. The board is slightly tilted in perspective, with a small metallic ball resting motionless inside a circular capture hole. Multiple other rounded holes are visible across the board, suggesting frequent capture points.

The pinball board appears **paused and dimmed**, with all motion frozen. No score counters, timers, or competitive UI elements are visible.

Centered on the screen is a **floating flashcard** with a white, rounded-rectangle background. The flashcard displays a **simple 3D fruit illustration** (for example: a shiny red apple) with **no text label**.

Above the flashcard, friendly rounded text reads:

**“Say it”**

Below the flashcard is a **microphone icon** surrounded by a softly pulsing circular ring, indicating that the game is listening for voice input.

Near the bottom of the screen is a **confidence meter** — a smooth horizontal bar that transitions from red to yellow to green and is partially filled. The meter feels encouraging and supportive rather than evaluative.

The overall aesthetic is **gentle, playful, and reassuring**, similar to a modern Montessori toy or an Apple-style educational app.

Visual characteristics:
- Soft shadows
- Pastel blues, greens, and warm fruit colors
- Rounded geometry
- Minimal visual clutter
- Friendly, accessible design

The scene communicates a moment of **quiet focus and active recall**, where the game is patiently waiting for the child to say the vocabulary word aloud.

---

## Optional Interaction Notes (If Supported)

- Device tilting subtly affects the board orientation during active gameplay.
- Touch-friendly bumpers and paddles are visible on the board but inactive during the recall state.
- Physics motion resumes smoothly after speech input.

---

## Optional Animation Notes (If Supported)

- The microphone pulse is slow, rhythmic, and calming.
- The confidence meter fills smoothly as speech is detected.
- Subtle glow effects appear when confidence increases.

---

## Design Intent Summary

This screen represents the **core learning moment** of the game:
- Gameplay pauses intentionally
- Audio is silent
- Visual focus is narrowed
- The learner is invited to actively recall and speak

The design prioritizes calm, confidence, and clarity over speed or competition.
