# Antriview — AI Voice Interview Platform

> An autonomous AI interviewer that conducts real-time voice interviews, generates adaptive questions based on candidate profile, and produces deep feedback reports with personalized learning roadmaps.

**Live demo:** `http://localhost:3000` · **Stack:** Next.js 15 · Firebase · Vapi AI · Google Gemini · TypeScript

---

## What It Does

Antriview replaces the static mock interview experience with a context-aware AI system that adapts to who the candidate is — their experience level, resume, and target role — before a single question is asked.

```
Candidate fills profile → AI generates adaptive questions → Voice interview (Vapi) →
Transcript analyzed → Gemini produces deep feedback → Personalized learning roadmap
```

The feedback is not a score card. It is a structured report with filler word detection, speaking pace analysis, identified skill gaps, a milestone-based learning plan, and course recommendations — all generated in a single Gemini call from the live transcript.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Next.js 15 App                      │
│  (App Router, React Server Components, Server Actions)  │
└────────────┬────────────────────────┬───────────────────┘
             │                        │
    ┌────────▼────────┐     ┌─────────▼──────────┐
    │  Firebase Auth  │     │  Firestore DB       │
    │  Session cookies│     │  users / interviews │
    │  Admin SDK      │     │  / feedback         │
    └─────────────────┘     └─────────────────────┘
             │                        │
    ┌────────▼────────┐     ┌─────────▼──────────┐
    │   Vapi AI SDK   │     │  Google Gemini      │
    │  Real-time STT  │     │  Question generation│
    │  TTS voice agent│     │  Feedback + roadmap │
    │  Transcript     │     │  Structured output  │
    └─────────────────┘     └────────────────────-┘
```

---

## Key Features

### 1. Adaptive Interview Generation
The system classifies candidates as Student, Fresher (0–2 years), or Professional (2+ years). Each type gets a different Gemini prompt, different scoring rubric, and a different estimated duration (15 / 30 / 45 minutes). Freshers and Professionals paste their resume and target JD — Gemini uses both to generate questions specific to their background.

```typescript
// Different focus instructions per user type
if (userType === 'student') {
  focusInstructions = `Focus on communication, academic projects,
  problem-solving approach. NOT deeply technical.`
} else if (userType === 'professional') {
  focusInstructions = `Leadership, system design, stakeholder management,
  strategic thinking. Challenging and in-depth.`
}
```

### 2. Real-Time Voice Interviews (Vapi)
Interviews run as live voice calls via the Vapi AI SDK. The assistant uses ElevenLabs TTS, Deepgram STT, and GPT-4 as the conversation model. The full transcript is captured and passed downstream for analysis.

### 3. Deep Feedback in a Single AI Call
After the interview, one Gemini call produces the entire report:

```typescript
const { object } = await generateObject({
  model: google('gemini-2.0-flash-001'),
  schema: enhancedFeedbackSchema,  // Zod-validated structured output
  prompt: buildFeedbackPrompt(transcript, userType, role, resumeText),
});
```

The schema enforces:
- Category scores with comments (different categories per user type)
- Filler word analysis (detected client-side from transcript before the call)
- Speaking pace and clarity rating
- Skill gaps as tagged items
- A learning plan with milestones, estimated hours, and resource links
- Course recommendations per skill gap (free and paid)

### 4. User-Type-Specific Scoring
Students are scored on Career Readiness and Learning Ability. Professionals are scored on Leadership & Impact and System Design. The scoring prompt is dynamically built from `getFeedbackCategories(userType)` so each report is relevant to the candidate's actual stage.

### 5. Firestore Data Model

```
users/{uid}
  └─ profile: { userType, targetRole, industry }

interviews/{id}
  └─ { role, level, techstack, questions[], userType, duration,
       resumeText, jdText, userId, createdAt }

feedback/{id}
  └─ { totalScore, categoryScores[], strengths[], areasForImprovement[],
       transcriptAnalysis, skillGaps[], learningPlan, courseRecommendations[] }
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 15.3 (App Router, Turbopack) |
| Language | TypeScript |
| Auth + DB | Firebase Auth + Firestore (Admin SDK server-side) |
| Voice AI | Vapi AI SDK — Deepgram STT, ElevenLabs TTS, GPT-4 |
| AI/LLM | Google Gemini 2.0 Flash via Vercel AI SDK |
| Schema validation | Zod (structured outputs) |
| UI | Tailwind CSS, shadcn/ui |
| Deployment | Vercel-ready |

---

## Project Structure

```
antriview/
├── app/
│   ├── (auth)/                  # Sign in / Sign up pages
│   ├── (root)/
│   │   ├── page.tsx             # Dashboard
│   │   ├── profile/page.tsx     # Profile setup (3-step wizard)
│   │   └── interview/
│   │       ├── page.tsx         # Interview generation
│   │       └── [id]/
│   │           ├── page.tsx     # Conduct interview (Vapi call)
│   │           └── feedback/    # Enhanced feedback report
│   └── api/vapi/generate/       # POST endpoint for Vapi → Gemini
├── components/
│   ├── Agent.tsx                # Voice call lifecycle + transcript capture
│   ├── InterviewGenerationForm.tsx   # Multi-step adaptive form
│   ├── InterviewGenerationFormWrapper.tsx  # Client wrapper
│   └── ProfileSetup.tsx         # User classification wizard
├── lib/
│   └── actions/
│       ├── auth.action.ts       # Auth + profile management
│       └── general.action.ts   # Interviews + enhanced feedback
└── constants/
    └── index.ts                 # Vapi config, Zod schemas, category logic
```

---

## How I Built This

This project was built using Claude as an AI coding partner — not as a code generator to paste blindly, but as a collaborator to decompose problems, review output, and maintain a high bar.

The workflow throughout:

1. **Decompose first.** Before writing any code, I mapped the full data flow: what gets collected, where it goes, what each API call needs to return. The architecture diagram above came before any implementation.

2. **One concern per change.** Each modification was scoped — update the Zod schema, then update the prompt, then update the Firestore write, then update the UI. Not all at once.

3. **Review AI output critically.** When Claude generated the feedback prompt, I read every instruction it was giving Gemini and adjusted the strictness, the fallback handling, and the filler word detection logic. The `countFillerWords` function was rewritten to use proper word-boundary regex after reviewing the first output.

4. **Test the data at each layer.** After each feature, I checked Firestore directly to verify the document had the right shape before moving to the UI layer.

5. **Keep the schema tight.** Using Zod `enhancedFeedbackSchema` with `generateObject` means if Gemini returns a malformed response, it fails loudly at the schema boundary — not silently in the UI.

---

## Setup

```bash
git clone https://github.com/your-username/antriview
cd antriview
npm install
```

Create `.env.local`:

```env
# Firebase Client
NEXT_PUBLIC_FIREBASE_API_KEY=
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=
NEXT_PUBLIC_FIREBASE_PROJECT_ID=
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=
NEXT_PUBLIC_FIREBASE_APP_ID=

# Firebase Admin
FIREBASE_PROJECT_ID=
FIREBASE_CLIENT_EMAIL=
FIREBASE_PRIVATE_KEY=""

# Vapi
NEXT_PUBLIC_VAPI_WEB_TOKEN=
NEXT_PUBLIC_VAPI_WORKFLOW_ID=

# Gemini
GOOGLE_GENERATIVE_AI_API_KEY=
```

```bash
npm run dev
```

---

## What's Next

The current version covers the full interview loop. The natural next layer is what SViam is building at a deeper level:

- **Context across rounds** — storing interview history per candidate and adjusting question difficulty and focus based on prior performance
- **Company-specific hiring bar** — letting companies define what "good" looks like for their role and calibrating the scoring rubric against their actual bar
- **Proctoring signals** — detecting when a candidate is reading from another screen, using unusual response patterns, or switching away during the interview
- **Coding interviews** — extending the voice layer with a live code editor and real-time evaluation of the solution

These are hard problems. The architecture here — Vapi for real-time voice, Gemini for structured reasoning, Firestore for fast read/write — is a solid foundation for all of them.

---

## About

Built as a demonstration of AI-native engineering: using AI tools to ship fast, staying in control of the output, and maintaining a high bar for what goes to production.

The goal was not to generate code. The goal was to build something that works.

---

*Built with Next.js, Firebase, Vapi AI, and Google Gemini.*