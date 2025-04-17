To keep this project — a **speaker‑aware, memory‑backed voice assistant built as a React PWA** — on track, the Product Requirements Document (PRD) below captures purpose, scope, user needs, functional specs, tech stack, milestones, and measurable success criteria.  

---

## 1  Overview & Purpose  

A Product Requirements Document defines **what** will be built, **why**, and the conditions for success; it aligns product, engineering, and stakeholders on a single source of truth citeturn0search0.  
This PRD covers the end‑to‑end assistant that:  

* Listens locally (SoftWhisper/Whisper.cpp)  
* Tags "who‑said‑what" (inaSpeechSegmenter)  
* Stores context in mem0 for personalization  
* Summarises with Gemini (Vertex AI)  
* Speaks back through a custom ElevenLabs voice  

All code will be authored in **Cursor IDE** and shipped as a browser‑installable PWA.

---

## 2  Goals & Success Metrics  

| Goal | Metric | Target |
|------|--------|--------|
| Zero paid STT cost | % minutes transcribed locally | 100 %  
| Low‑latency captions | End‑to‑end delay (speech→UI) | ≤ 1 s on `tiny.en`  
| Accurate speaker labels | Diarization Error Rate | ≤ 20 % on two‑speaker test set citeturn0search7  
| Personalised responses | Recall of prior facts in summary | > 80 % of test prompts  
| Voice naturalness | MOS rating of ElevenLabs output | ≥ 4 / 5  

---

## 3  Background & Rationale  

* **SoftWhisper April 2025** adds built‑in speaker identification, eliminating the need for external diarization services citeturn0search1.  
* Whisper.cpp supports a `--rt` flag plus stdin piping, enabling near‑real‑time transcription inside Node wrappers citeturn0search2.  
* **Gemini 2.0** is available through the Vertex AI Node SDK, making server‑side LLM integration straightforward in JavaScript citeturn0search4.  
* **ElevenLabs** allows API‑based custom‑voice cloning; each voice is addressed by a `voice_id` retrievable from the dashboard or API citeturn0search3.  
* **mem0** offers a purpose‑built, vector‑backed "memory layer" to persist user interactions and feed them back into prompts citeturn0search5.  
* Shipping as a PWA grants offline installability and mobile‑like UX once manifest and service‑worker criteria are met citeturn0search6.  

---

## 4  Target Users  

* Tech‑savvy professionals who prefer local processing for privacy and cost control  
* Users who consume long‑form meetings/podcasts and need fast diarized transcripts  
* Early adopters comfortable installing PWAs from desktop or mobile browsers

---

## 5  Assumptions & Constraints  

* Users have browsers supporting **MediaRecorder** and **WebAssembly**.  
* Local CPU/GPU resources suffice for at least the `tiny.en` Whisper model.  
* ElevenLabs TTS quotas cover expected daily usage.  
* All secrets (Vertex, mem0, ElevenLabs) will be managed via environment variables / Docker secrets.  

---

## 6  Functional Requirements  

### 6.1 GUI (React PWA)  

* **Controls**: Record / Stop / Play / "Generate Summary" buttons.  
* **Live captions**: Streamed JSON segments rendered in a scrollable list.  
* **Install prompt**: Trigger `beforeinstallprompt` when manifest criteria satisfied citeturn0search6.  
* **Media Capture**: Use `navigator.mediaDevices.getUserMedia()` and `MediaRecorder` to produce ~100 ms audio chunks citeturn1search0.  

### 6.2 STT & Diarization Layer  

* Spawn SoftWhisper with `--model medium --diarize --json --rt -`.  
* Accept raw 16‑kHz PCM from the WebSocket; emit JSON `{text, speaker, start, end}`.  
* Support fallback to `tiny.en` for constrained devices.  
* Future enhancement: adopt Whisper Speaker Identification (WSI) embeddings for higher accuracy citeturn0search8.  

### 6.3 Memory Layer (mem0)  

* Persist finalised transcript and Gemini summary per session.  
* Retrieve top‑K relevant memories for each new prompt to Gemini.  

### 6.4 LLM Summarisation (Gemini)  

* Endpoint `/summarize` receives full transcript + top memories.  
* Prompt template:  
  ```
  Context: {mem0_results}
  Transcript: {transcript}
  Summarise in ≤150 words
  ```  
* Return streaming chunks to front‑end for progressive rendering.  

### 6.5 Text‑to‑Speech (ElevenLabs)  

* Call `/v1/text-to-speech/{voice_id}` with `model_id=eleven_turbo_v2_5` for lowest latency citeturn0search3.  
* Stream MP3 back to a hidden `<audio>` element.  

### 6.6 Data Privacy & Security  

* All raw audio remains local; only text is sent to cloud LLM.  
* Front‑end served over HTTPS; WebSockets upgraded to WSS.  
* CSP forbids inline scripts.  

---

## 7  Non‑Functional Requirements  

| Category | Requirement |
|----------|-------------|
| Performance | ≤1 s caption latency; ≤800 ms TTS playback start. |
| Scalability | Docker Compose allows horizontal scaling of Gemini proxy. |
| Availability | Target 99.5 % uptime with automated restarts. |
| Accessibility | Captions follow WCAG 2.2 colour‑contrast ratios. |
| Maintainability | 80 % unit‑test coverage across API layer. |

---

## 8  Tech Stack & Integration Architecture  

| Layer | Technology |
|-------|------------|
| Front‑end | React 18, Vite, TypeScript, TailwindCSS |
| Service‑worker | Workbox offline strategies |
| STT wrapper | Node 20 (WS server) + Python 3.11 SoftWhisper |
| Diarization | `inaSpeechSegmenter` citeturn0search7 |
| Memory | mem0 (cloud or self‑host Docker) |
| LLM | Gemini 2.0 via Vertex AI SDK citeturn0search4 |
| TTS | ElevenLabs API citeturn0search3 |
| Containerisation | Docker Compose (web, api, stt, mem0) |

---

## 9  Milestones & Timeline  

| Date | Deliverable |
|------|-------------|
| Week 1 | Repo scaffolding in Cursor, PWA skeleton, Docker dev‑container |
| Week 2 | STT wrapper + SoftWhisper integration |
| Week 3 | Live caption UI, diarization colour‑coding |
| Week 4 | mem0 persistence & Gemini summary endpoint |
| Week 5 | ElevenLabs playback, accessibility polish |
| Week 6 | Security hardening, load testing, beta release |

---

## 10  Out‑of‑Scope  

* Real‑time translation into other languages  
* Cloud‑hosted STT services (Deepgram, AssemblyAI)  
* Mobile native wrappers (Capacitor) for initial launch  

---

## 11  Risks & Mitigations  

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Whisper model too heavy for low‑end CPUs | Medium | High | Offer `tiny.en` auto‑fallback and GPU build option. |
| ElevenLabs quota overrun | Low | Medium | Cache TTS MP3 and reuse; monitor usage via API. |
| Memory retrieval degrades with scale | Medium | Medium | Nightly prune & vector‑reindex in mem0. |

---

## 12  Acceptance Criteria  

* User can install PWA on Chrome → `beforeinstallprompt` fires citeturn0search6.  
* Speaking into mic shows captions within one second.  
* Clicking "Generate Summary" returns a Gemini response that references at least one past memory item.  
* "Play Voice" button outputs ElevenLabs audio identical in text to Gemini summary.  

---

## 13  Appendix & Reference Links  

1. Atlassian PRD definition citeturn0search0  
2. SoftWhisper April 2025 release citeturn0search1  
3. Whisper.cpp real‑time flag discussion citeturn0search2  
4. Vertex AI Node quick‑start citeturn0search4  
5. ElevenLabs voice ID guide citeturn0search3  
6. mem0 product site citeturn0search5  
7. PWA installability criteria citeturn0search6  
8. MediaRecorder usage citeturn1search0  
9. inaSpeechSegmenter example citeturn0search7  
10. WSI speaker‑ID roadmap citeturn0search8  

---

**End of PRD.** Feel free to iterate on scope or milestones, and let me know when you're ready to translate this into user stories or technical specs. 