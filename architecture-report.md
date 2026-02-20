# App Architecture Report: Voice Questionnaire

## 1. Overview (Fill in manually)

- **App Name**: Voice Questionnaire
- **Purpose / Description**: AI-validated deterministic voice questionnaire for insurance applications. Uses voice input/output to walk applicants through a medical/underwriting questionnaire with AI-powered answer validation and optional LLM follow-up questions.
- **Primary Users**: Protective stakeholders involved in demo
- **Access Scope**: [UNKNOWN — needs manual input]
- **Built By / Maintained By**: Emory Wise
- **First Deployed**: Dec 19, 2025
- **Current Status**: Inactive - Passed to Protective

---

## 2. Tech Stack

### Frontend
- **Framework / Library**: Vanilla JavaScript (no framework) — single-file SPA in `public/index.html` (2,013 lines)
- **Language**: JavaScript (ES6+)
- **Styling**: Tailwind CSS via CDN (`https://cdn.tailwindcss.com`) + custom CSS variables for brand theming
- **Build Tool**: None — no build step; frontend is a single HTML file served directly by Express
- **Key Dependencies** (all browser-native or CDN):
  - **Tailwind CSS** (CDN) — utility-first CSS framework
  - **Web Speech API** (browser-native) — `SpeechRecognition` / `webkitSpeechRecognition` for voice input
  - **Web Speech Synthesis** (browser-native) — `speechSynthesis` for fallback TTS
  - **Web Audio API** (browser-native) — `AudioContext` for ElevenLabs audio playback
- **Notable Frontend Classes**:
  - `TTSService` — Text-to-speech (ElevenLabs via backend, fallback to Web Speech)
  - `ASRService` — Speech recognition (Web Speech API)
  - `ValidationService` — Calls backend `/api/validate`
  - `WhyService` — Calls backend `/api/why`
  - `FollowupService` — Calls backend `/api/followup` + `/api/followup-check`
  - `FlowController` — State machine managing deterministic question flow
  - `render()` — UI rendering function (template literals, no virtual DOM)

### Backend
- **Framework**: Express.js 4.18.2
- **Language / Runtime**: Node.js >= 18 (JavaScript, CommonJS modules)
- **Key Dependencies** (from `package.json`):
  - `express` ^4.18.2 — HTTP server and routing
  - `cors` ^2.8.5 — Cross-origin resource sharing middleware
  - `dotenv` ^16.3.1 — Environment variable loading
- **Architecture**: Monolithic single-file backend (`server.js`, 817 lines). Acts primarily as an API proxy — keeps API keys server-side and forwards requests to external LLM/TTS services.

### Database(s)
- **Type & Engine**: [N/A] — No database. All state is held in-memory on the frontend (browser).
- **Hosting**: [N/A]
- **ORM / Query Layer**: [N/A]
- **Migration Tool**: [N/A]
- **Connection Pattern**: [N/A]
- **Schema Notes**: Question definitions are a static JavaScript array (`QUESTIONS`) embedded in `public/index.html`. An external JSON file (`protective_underwriting_decision_tree (1).json`) contains a reference decision tree schema but is not loaded by the application at runtime.

---

## 3. Authentication & Authorization

### Authentication
- **Method**: None — no authentication is implemented
- **Flow**: [N/A] — the application is open to anyone who can access the URL
- **Session Management**: [N/A] — no sessions; all state is ephemeral in-browser
- **Domain/Tenant Restriction**: [N/A]

### Authorization
- **Model**: None — no authorization beyond network access
- **Roles Defined**: [N/A]
- **How Enforced**: [N/A]

---

## 4. External Services & Integrations

| Service | Purpose | Direction | How Connected | Auth Method |
|---------|---------|-----------|---------------|-------------|
| **ElevenLabs** | High-quality text-to-speech (voice output) | Outbound (backend → ElevenLabs) | REST API via `https://api.elevenlabs.io/v1/text-to-speech/{voiceId}/stream` | API key in `xi-api-key` header (from `ELEVENLABS_API_KEY` env var) |
| **Anthropic Claude** | Answer validation, "why" explanations, follow-up question generation | Outbound (backend → Anthropic) | REST API via `https://api.anthropic.com/v1/messages` | API key in `x-api-key` header (from `ANTHROPIC_API_KEY` env var) |
| **OpenAI** | Alternative provider for validation, explanations, follow-ups | Outbound (backend → OpenAI) | REST API via `https://api.openai.com/v1/chat/completions` | Bearer token (from `OPENAI_API_KEY` env var) |
| **Web Speech API** | Voice recognition (speech-to-text) | Browser-native | Browser API (`SpeechRecognition`) — no backend involvement | None (browser permission) |
| **Web Speech Synthesis** | Fallback text-to-speech | Browser-native | Browser API (`speechSynthesis`) — no backend involvement | None |

**Notes**:
- Anthropic uses model `claude-sonnet-4-20250514`; OpenAI uses model `gpt-4o-mini`
- ElevenLabs uses model `eleven_turbo_v2_5` with default voice ID `EXAVITQu4vr4xnSDxMaL`
- The `VALIDATION_PROVIDER` env var controls whether Anthropic or OpenAI is used (defaults to `anthropic`)
- All external API calls are proxied through the Express backend — no API keys are exposed to the browser
- The app includes complete fallback behavior: rule-based validation when no LLM API keys are configured, Web Speech fallback when ElevenLabs is unavailable

---

## 5. Additional Infrastructure Patterns

- **Background Jobs / Cron**: [None detected]
- **File Storage**: [None detected] — no file uploads or persistent file storage
- **WebSocket / Real-Time**: [None detected] — all communication is standard HTTP request/response
- **Caching**: [None detected] — no caching layer; every validation/TTS call hits the external API
- **Message Queues / Event Buses**: [None detected]
- **Static Assets / CDN**: Tailwind CSS loaded from `cdn.tailwindcss.com`. A `logo.svg` file exists in `public/`. All other assets are inline in the HTML file.

---

## 6. Environment Variables & Secrets

| Variable Name | Category | Description | Where Used |
|---------------|----------|-------------|------------|
| `ELEVENLABS_API_KEY` | External Service | ElevenLabs API key for text-to-speech | `server.js` — TTS endpoint, config check |
| `ELEVENLABS_VOICE_ID` | App Config | ElevenLabs voice ID (default: `EXAVITQu4vr4xnSDxMaL`) | `server.js` — TTS endpoint URL |
| `ANTHROPIC_API_KEY` | External Service | Anthropic Claude API key for validation/followups/explanations | `server.js` — validation, followup, why endpoints |
| `OPENAI_API_KEY` | External Service | OpenAI API key (alternative to Anthropic) | `server.js` — validation, followup, why endpoints |
| `VALIDATION_PROVIDER` | App Config | Which LLM provider to use: `anthropic` or `openai` (default: `anthropic`) | `server.js` — routing logic for all LLM calls |
| `PORT` | App Config | Server port (default: `3000`) | `server.js` — Express listen |

**Notes**: No secrets are hardcoded. All sensitive values come from environment variables loaded via `dotenv`.

---

## 7. Deployment & Infrastructure

- **Hosting Platform**: [UNKNOWN — needs manual input] *(Dockerfile present suggests containerized deployment; README mentions Heroku, Railway as examples; no `render.yaml` or platform-specific config found)*
- **Deployment Method**: Docker container (`Dockerfile` present) or direct Node.js deployment
- **Deployment Config Files**:
  - `Dockerfile` — Node.js 18 Alpine image, runs `npm ci --omit=dev`, exposes port 3000
  - `.dockerignore` — (not inspected separately, but present)
- **Build & Start Commands**:
  - **Build**: `npm ci --omit=dev` (no frontend build step)
  - **Start**: `npm start` → `node server.js`
  - **Dev**: `npm run dev` → `node --watch server.js`
- **Environment(s)**: [UNKNOWN — needs manual input] *(No evidence of staging/production environment separation in code)*
- **Domain / URL**: [UNKNOWN — needs manual input]
- **SSL/TLS**: [UNKNOWN — needs manual input] *(No SSL configuration in app code; likely handled by hosting platform/reverse proxy)*

### Architecture Pattern
- **Monolith or Separate Services**: Monolithic single deploy — one Express server serves both the static frontend and the API
- **How Frontend is Served**: Static files served by Express via `express.static(path.join(__dirname, 'public'))`. No SSR, no build step. The frontend is a single `index.html` file with all JavaScript inline.

---

## 8. Deployment Readiness Assessment

- [ ] **Health Check Endpoint**: No `/health` or similar endpoint detected. The server has no health check route.
- [x] **Error Handling**: Backend has try/catch on all endpoints with fallback responses. Frontend FlowController has error state handling. No global Express error handler middleware though.
- [ ] **Logging**: Only `console.log` and `console.error` — no structured logging library (no Winston, Pino, etc.)
- [ ] **Monitoring / Alerting**: No APM or monitoring configured (no Sentry, Datadog, etc.)
- [x] **Secrets Management**: All secrets properly externalized via environment variables with `dotenv`. No hardcoded secrets found.
- [ ] **Input Validation**: Basic presence checks on required fields (`!question || !questionType`), but no schema validation library (no Zod, Joi, etc.). The `text` field for TTS is passed directly to external APIs without sanitization.
- [ ] **Rate Limiting**: No rate limiting on any endpoint. External API calls (ElevenLabs, Anthropic, OpenAI) could be abused.
- [ ] **CORS Configuration**: CORS is configured as wide-open wildcard — `app.use(cors())` with no origin restrictions.
- [ ] **Database Backups**: [N/A] — no database
- [x] **Dependency Security**: 1 high-severity vulnerability found (`qs` package — DoS via memory exhaustion, transitive dependency of Express). Fixable via `npm audit fix`.
- [x] **README / Documentation**: Comprehensive README with setup instructions, API docs, troubleshooting, and project structure. AGENTS.md provides AI assistant context.
- [ ] **Environment Separation**: No distinct dev/staging/production configs detected. Single `.env.example` for all environments.

**Additional Notes**:
- The frontend uses Tailwind CSS from CDN (`cdn.tailwindcss.com`) — this is not recommended for production (slower load times, no tree-shaking). Should use a build step with Tailwind CLI or PostCSS.
- The `public/index.html` is 2,013 lines and ~79KB — a very large single file containing all frontend logic, styles, and templates. This works but makes maintenance harder.
- A `public/index.backup.html` file exists (56KB), suggesting manual backup practices rather than version-controlled branching.
- The `protective_underwriting_decision_tree (1).json` file in the repo root appears to be a reference/data file not loaded by the app. The filename contains spaces and parentheses.
- Language support (English + Spanish) exists but the language selector is hidden behind a feature flag (`SHOW_LANGUAGE_SELECTOR = false`).
- No authentication means the API endpoints are publicly accessible — anyone could call `/api/validate` or `/api/tts` and consume API credits.
- No global Express error handler — unhandled errors in middleware could crash the process.

---
