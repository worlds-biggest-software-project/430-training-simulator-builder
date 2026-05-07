# Training Simulator Builder

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-powered authoring platform that enables subject-matter experts to build realistic, branching training simulations for compliance and skills development -- without specialized instructional design skills or weeks of production time.

Training Simulator Builder is a scenario-based simulation authoring tool for corporate L&D teams, compliance officers, and instructional designers in regulated industries. It addresses the central bottleneck in simulation training: the time, cost, and technical skill required to produce branching scenarios with realistic dialogue, consequence logic, and scoring. By combining a visual authoring interface with AI-assisted content generation, it compresses simulation production from weeks to hours and makes the format accessible to non-technical authors.

---

## Why Training Simulator Builder?

- **Existing tools are expensive and enterprise-only.** Articulate Storyline 360 costs ~$1,749/user/year, Talespin is sales-led enterprise pricing, and most AI simulation platforms (SymTrain, Virti, Mindtickle) target narrow verticals with closed ecosystems. Mid-market teams and freelance instructional designers are priced out.
- **General authoring tools lack AI-native simulation features.** Storyline and Adobe Captivate require manual construction of every branch, trigger, and variable. Building a compliance scenario with dozens of decision points takes weeks of skilled authoring effort.
- **Purpose-built simulation tools are vertically locked.** SymTrain serves contact centers. Mindtickle serves B2B sales. Virti is healthcare-focused. No single platform covers the breadth of compliance and skills training scenarios that organizations need.
- **No open-source option exists with full capability.** H5P offers a branching scenario content type under MIT licence, but it lacks AI authoring assistance, character libraries, voice synthesis, and analytics dashboards. There is no open-source foundation that can be extended and self-hosted by organizations with data privacy constraints.
- **AI-generated content is generic, not policy-accurate.** Current AI tools generate plausible dialogue but cannot ingest an organization's actual policies to produce legally defensible compliance scenarios. A harassment training simulation must reflect the company's specific policy, not a generic script.

---

## Key Features

### Visual Scenario Authoring

- Node-graph branching editor for creating decision points, character responses, and consequence paths without code
- Support for complex branching logic including multiple paths, converging threads, and looping
- Consequence and scoring logic with points for correct choices and feedback on incorrect paths
- Post-scenario debrief with per-path feedback messages

### AI-Assisted Content Generation

- Generate realistic dialogue and scenario branches from a brief training objective or uploaded policy document
- AI-suggested response options and consequence paths during authoring
- Dialogue quality review that flags unrealistic, legally inaccurate, or policy-inconsistent content
- Automatic scenario variation generation from a master scenario

### Character and Voice System

- Configurable character library with customizable appearance and voice profiles
- Text-to-speech voice generation via API integration (ElevenLabs or equivalent)
- Multi-character interaction support for realistic scenario dynamics

### Compliance and Skills Templates

- Pre-built scenario structures for workplace harassment, workplace safety, data privacy, and code of conduct
- Additional template packs for sales conversation, clinical communication, and leadership/coaching
- Templates customizable with organization-specific policies and context

### LMS Integration and Standards

- SCORM 1.2, SCORM 2004, and xAPI export for deployment to any compatible LMS
- cmi5 export for environments supporting the newer standard
- LTI 1.3 deep integration for native LMS embedding
- Hosted learner delivery with completion tracking for organizations without an LMS

### Analytics and Collaboration

- Learner analytics dashboard: path frequency, decision-point error rates, completion time
- Multi-author collaboration with role-based access (author, reviewer, publisher) and comment threads
- Version history for scenario content
- Regulatory evidence reports linking simulation completion to specific compliance requirements

---

## AI-Native Advantage

The core production bottleneck in simulation training is authoring: manually writing realistic dialogue for every branch, anticipating learner responses, and constructing consequence logic. AI generation compresses this from weeks to hours by producing policy-grounded dialogue from uploaded documents, suggesting branching paths the author may not have considered, and evaluating free-text learner responses semantically rather than requiring rigid branching trees. This enables an open-ended conversational practice mode where learners type or speak naturally and the system assesses their response -- a capability currently limited to narrow verticals like contact center training.

---

## Tech Stack & Deployment

- **Deployment modes:** Self-hosted (for organizations with data privacy constraints) and cloud-hosted
- **Open standards:** SCORM 1.2, SCORM 2004, xAPI (Tin Can), cmi5, LTI 1.3 -- all open specifications with no patent encumbrances
- **AI integration:** LLM APIs for dialogue generation and semantic response evaluation; TTS APIs for character voice synthesis
- **Learner experience:** HTML5 responsive output for desktop and mobile; touch-friendly interaction patterns
- **Accessibility target:** WCAG 2.2 Level AA for learner-facing output

---

## Market Context

The global eLearning market is projected to exceed $400 billion by 2030, with simulation training growing faster than the overall market as organizations recognize the limits of passive content for behavioral change. Corporate compliance training alone represents a multi-billion-dollar annual spend driven by regulatory mandates across financial services, healthcare, manufacturing, and energy. Incumbent authoring tools range from ~$499/year (Rise 360) to ~$1,749/year (Storyline 360) per author seat, with enterprise simulation platforms like Talespin and Virti priced significantly higher on custom contracts. Primary buyers are corporate L&D teams in regulated industries, HR compliance functions, and eLearning agencies building simulation content for enterprise clients.

---

## Project Status

> This project is in the **research and specification phase**.
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
