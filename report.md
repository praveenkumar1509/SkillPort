# Skill-Port Project — Complete Technical Documentation

## 1. Project Architecture Overview

The **Skill-Port** platform is a sophisticated, AI-powered career coaching ecosystem designed to bridge the gap between academic preparation and industry requirements. Structurally, the project is a modern Single Page Application (SPA) built using **React 18+**, **Vite**, **TypeScript**, and **Tailwind CSS**. Its architecture is uniquely characterized by a hybrid, dual-Supabase backend strategy that balances legacy stability with modern feature scalability.

The project follows a modular "feature-first" directory structure under `src/modules` and `src/pages`. This separation ensures that complex business logic (like AI interviewing and GitHub verification) is encapsulated within specific modules while being consumed by high-level page components. The UI is built on a premium "card-based" layout, utilizing **Framer Motion** for micro-animations and **Lucide React** for consistent iconography.

## 2. Authentication & Role System

The Authentication and Role system is the gateway to the platform, defining two distinct user experiences: **Students** (Job Seekers) and **Companies** (Recruiters).

- **Technical Implementation**: Built on **Supabase Auth**, the system uses a custom `AuthContext` (`src/contexts/AuthContext.tsx`) to manage session state globally.
- **Role Assignment**: During signup (`SignupPage.tsx`), users select their role (`student` or `company`). This role is stored in two locations for redundancy and performance:
    1.  **User Metadata**: ` freshUser?.user_metadata?.userRole` enables immediate redirection post-auth without extra DB queries.
    2.  **Profiles Table**: Stored within `resume_data.userRole` in the managed Supabase instance.
- **Redirection Logic**: The `LoginPage.tsx` implements a robust redirect handler. Upon successful sign-in, it interrogates the metadata. If missing, it falls back to a database lookup. 
    - **Company users** are directed to `/company-dashboard`.
    - **Student users** are check for profile completeness (existence of `career_goal` or `skills`). If incomplete, they are routed to `/profile-setup`; otherwise, to the `/dashboard`.
- **Key Decision**: Using a combination of OAuth metadata and database profiles resolves potential race conditions during account creation where a profile might not yet be initialized by Supabase triggers.

## 3. Profile Setup

The Profile Setup module (`ProfileSetupPage.tsx`) is designed to capture core career attributes early in the user lifecycle.

- **Purpose**: It serves as the "onboarding" phase for students, collecting the `career_goal`, `primary_skills`, and `experience_years`.
- **Data Model**: This data is persisted in the `profiles` table of the managed Supabase instance.
- **State Management**: Local state manages form inputs, while the `useAuth` hook is used to refresh the global profile state after updates, ensuring the dashboard immediately reflects the new information.
- **Validation**: Strict validation ensures that students cannot proceed to the dashboard without a defined career goal, which is essential for the AI-driven modules (Skill Gap and Interview) to function correctly.

## 4. Student Dashboard

The Student Dashboard (`DashboardPage.tsx`) acts as the central command center for the job seeker.

- **Implementation**: It is a real-time reactive interface that aggregates data from both Supabase instances.
- **Features**:
    - **Job Recommendations**: Fetches active jobs from the `jobs` table in the dedicated Supabase instance (`companySupabase`).
    - **Application Tracking**: Monitors the status of submitted applications.
    - **Module Shortcuts**: Direct entry points to Resume Parsing, GitHub Verification, and AI Interviewing.
- **Real-time Sync**: Uses `subscribeToActiveJobs` from `companyApi.ts` to listen for new job postings or status changes via Postgres CDC (Change Data Capture), allowing students to see new opportunities without refreshing.

## 5. Resume Parser

A cornerstone of the platform, the Resume Parser (`ResumePage.tsx`) transforms unstructured PDF/DOCX files into structured JSON data.

- **Tech Stack**:
    - **PDF.js**: For standard PDF text extraction.
    - **Mammoth.js**: Specifically included for high-fidelity DOCX conversion.
    - **Tesseract.js**: Integrated as an **OCR fallback** for scanned resumes. If the extracted text length is `< 300` characters, the system automatically renders PDF pages to a canvas and runs optical character recognition.
- **AI Integration**: The raw text is sent to the `ai-assistant` Supabase Edge Function, which uses **Gemini** to extract skills, education, experience, and a professional summary.
- **Custom Solution**: The worker configuration for PDF.js is handled locally (`pdf.worker.min.mjs`) to avoid external CDN dependencies and potential CORS issues in production environments.
- **Persistence**: Structured resume data is stored in the `resumes` table of the user's dedicated Supabase instance, while extracted skills are synced back to the primary `profiles` table for platform-wide use.

## 6. GitHub Verification Module

The GitHub Verification Module (`src/modules/github/GitHubVerification.tsx`) provides an objective assessment of a student's technical repertoire.

- **Auth Mechanisms**: Supports **GitHub Device Flow** and **Personal Access Tokens (PAT)**. The Device Flow is preferred for UX, while PAT (Classic only, as fine-grained tokens are restricted in GraphQL) serves as a fallback for complex repo access.
- **Architecture**:
    - **GraphQL API**: A single, complex query fetches repository metadata and the last 30 commits (commit messages, authors, dates). This is significantly more efficient than multiple REST calls.
    - **REST API Fallback**: Used specifically for fetching critical files like `package.json` and `README.md` to identify the tech stack and project documentation quality.
- **AI Evaluation**: Commit history and manifest contents are passed to Gemini via `geminiCall`. The AI generates a structured JSON review covering code quality, tech stack, strengths, and suggestions.
- **Authenticity Scoring**: A custom algorithm calculates an "Authenticity Score" based on commit dominance (how much of the code was committed by the user vs. others), providing a transparency layer for recruiters.

## 7. Skill Gap Analysis

The Skill Gap Analysis module (`SkillGapPage.tsx`) uses AI to provide a roadmap for career advancement.

- **Purpose**: It compares the user's "Current Skills" (sourced from Resume + GitHub) against their "Career Goal".
- **Logic**: It aggregates data from the `resumes` and `github_verifications` tables.
- **AI Processing**: Gemini acts as a "Senior Career Coach," identifying missing competencies, rating current proficiency levels, and generating a step-by-step learning roadmap.
- **Outputs**:
    - **Priority Score**: A numerical indicator (0-100) of how critical the gap is.
    - **Visual Gaps**: Progress bars showing `Current` vs. `Needed` levels.
    - **Roadmap**: Actionable steps (e.g., "Step 1: Learn Redux for state management").
- **Storage**: Results are saved to the `skill_gaps` table for persistence across sessions.

## 8. AI Interview Simulator

The most technically complex module, the AI Interview Simulator (`InterviewPage.tsx`), provides a high-fidelity practice environment.

- **Core Technologies**:
    - **Web Speech API**: Handles both `webkitSpeechRecognition` (STT) and `speechSynthesis` (TTS).
    - **Web Audio API**: An `AudioContext` is utilized to create a 3x gain boost for the microphone, ensuring reliable transcription even in noisier environments.
- **Custom Features**:
    - **Naturalness Scoring**: A specialized hook `useNaturalSpeech` analyzes speech rate, filler words (um, ah, like), and hesitation gaps.
    - **Auto-Listen Workflow**: The AI speaks the question (TTS), and the microphone automatically activates upon completion.
    - **Silence Detection**: A `2.5s` silence timer triggers an automatic submission of the answer, simulating the flow of a real conversation.
- **Scoring Engine**: Gemini evaluates each answer for technical accuracy and communication clarity. An "Industry Readiness Score" is then computed by blending the interview score, naturalness score, resume quality, and GitHub verification results.
- **Multi-Instance Storage**: Global results and specific Q&A pairs are persisted in the `interviews` and `feedbacks` tables on the dedicated Supabase instance.

## 9. Recruiter/Company Side

The recruiter experience is a premium SaaS dashboard designed for high-volume talent discovery.

- **Company Dashboard**: A command center featuring real-time stats (Total Candidates, Active Jobs) and a skill-based search engine.
- **Job Postings**: CRUD interface for managing recruitment needs. Jobs are stored in the dedicated Supabase instance and made public to the student dashboard.
- **Applications**: A centralized list for reviewing candidate submissions. Recruiters can approve/reject candidates, which triggers real-time status updates for the student.
- **Candidate Search**: A deep search utility that filters the `profiles` table based on specific skill requirements. It utilizes the unified skill set (Extracted + Verified).
- **Company Profile**: A dedicated branding page (`CompanyProfilePage.tsx`) for employers to showcase their bio, industry, and logo, implemented with an "Employer Branding" aesthetic (glassmorphism and teal accents).
- **Analytics**: Built with **Recharts**, it provides visual insights into the hiring funnel, including weekly application trends and candidate status distribution.

## 10. Multi-Supabase Architecture & Data Flow

Skill-Port employs a strategic **Split-Backend Architecture**:

1.  **Managed Supabase (Primary)**:
    - **Tables**: `profiles`, `auth.users`.
    - **Purpose**: Core identity management and shared user profiles. This is the "Shared Source of Truth" for authentication.
2.  **User-Dedicated Supabase (Feature)**:
    - **Tables**: `jobs`, `applications`, `resumes`, `github_verifications`, `interviews`, `skill_gaps`, `feedbacks`, `companies`.
    - **Purpose**: Housing all domain-specific data and modern enterprise features.
- **Communication Layer**: The `companyApi.ts` acts as the bridge, using two separate clients (`supabase` and `companySupabase`) to perform cross-instance data synchronization. For example, search queries filter the Managed instance but saved results are stored in the Dedicated instance.

## 11. Real-time Synchronization Strategy

The platform maintains state across modules using **Postgres Change Data Capture (CDC)** via Supabase Channels.

- **Recruiter Sync**: Recruiter dashboards listen to the `jobs` and `applications` tables. When a student applies, the `applications` table insert triggers a real-time event that updates the recruiter's "Pending" count without a refresh.
- **Student Sync**: Students subscribe to their specific `user_id` in the `applications` table. Recruiter status changes (e.g., "Screening" to "Interviewing") are pushed instantly to the student's dashboard.
- **Implementation**: Handled through `subscribeToRecruiterEvents` and `subscribeToStudentApplications` utilities, which manage channel lifecycle (subscription on mount, teardown on unmount).

## 12. UI/UX Implementation & Theme Consistency

The visual identity of Skill-Port is defined by a **Premium Light Theme**:

- **Color Palette**: 
    - **Primary**: Teal (`#0f766e`) for action buttons and accents.
    - **Secondary**: Lavender (`#d6bcfa`) and soft grays (`#f8fafc`) for backgrounds and cards.
- **Typography**: Modern sans-serif stack utilizing Inter or system defaults, optimized for readability in data-heavy modules.
- **Components**: Standardized using **Shadcn UI** primitives (Cards, Dialogs, Badges), customized with premium box-shadows (`shadow-premium`) and border-radii (`rounded-2xl`).
- **Animations**: **Framer Motion** powers "Page Transitions" (fade-in/slide-up) and "Micro-interactions" (button lifts, hover scales), ensuring the app feels responsive and "alive."

---

## Overall Strengths and Future Recommendations

### Strengths
- **Sophisticated AI Orchestration**: Seamless integration of Gemini for resume parsing, code review, and interview coaching.
- **Superior Speech Tech**: Custom mic boost and silence detection provide a significantly better interview experience than standard STT implementations.
- **Scalable Architecture**: The dual-Supabase setup allows for isolated growth of enterprise recruiter features without disturbing the core student auth flow.
- **Data Integrity**: Unified skill extraction from both static documents (Resumes) and dynamic evidence (GitHub) creates a highly accurate candidate profile.

### Future Recommendations
1.  **Advanced Filtering**: Implement vector search (Supabase pgvector) for more semantic candidate matching (e.g., searching for "Expert in modern frontend" instead of just "React").
2.  **White-Labeling**: Enhance the Company Profile with more customization (custom colors/banners) to support stronger employer branding.
3.  **Video Analysis**: Supplement the voice-based interview with basic facial expression or posture analysis using MediaPipe.
4.  **Multi-Instance Migration**: Consider consolidating into a single Supabase project once the "legacy" constraints are lifted, reducing client-side complexity.

---
*Documentation Version: 1.0.0*
*Architect: Senior Technical Architect & Documentation Expert*
