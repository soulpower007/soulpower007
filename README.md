<p align="center">
  <a href="https://livekit.io/">
    <img src="./.github/assets/livekit-mark.png" alt="LiveKit logo" width="96" height="96">
  </a>
</p>

<h1 align="center">Post-Discharge Voice Agent</h1>

<p align="center">
  A LiveKit-powered voice AI care coordinator for hospital discharge check-in calls.
</p>

<p align="center">
  <a href="https://github.com/livekit/agents"><img alt="LiveKit Agents" src="https://img.shields.io/badge/LiveKit-Agents-00AEEF"></a>
  <a href="https://docs.astral.sh/uv/"><img alt="uv" src="https://img.shields.io/badge/package%20manager-uv-2D2D2D"></a>
  <img alt="Python" src="https://img.shields.io/badge/python-3.10--3.13-blue">
  <img alt="License" src="https://img.shields.io/badge/license-MIT-green">
</p>

## The Gist

This project is a realistic healthcare voice-agent prototype that calls a patient after hospital discharge, walks through a structured check-in, catches safety issues, and produces a nurse-ready summary.

It is built with [LiveKit Agents](https://github.com/livekit/agents), LiveKit Cloud, and LiveKit Inference. The interesting part is the architecture: instead of one giant prompt, the call is split into a coordinator agent, scoped intake tasks, a hard escalation handoff, and a background safety observer.

> This is a demo/prototype for care-coordination workflows. It does not diagnose, prescribe, or replace clinical judgment.

## Why It Is Cool

- Real-time voice intake for a high-stakes healthcare workflow.
- A six-step discharge checklist: identity, safety, medications, follow-up, equipment/homecare, and caregiver support.
- Immediate nurse escalation when red-flag symptoms appear.
- Background medical-policy monitoring that watches every user turn without blocking the conversation.
- Guardrails for medication advice and diagnosis attempts.
- Mock backend tools for scheduling, DME/homecare coordination, nurse escalation, and nurse-note generation.
- Typed `CallState` shared across agents, tasks, tools, and summaries.
- Behavior-focused tests for handoffs, tools, guardrails, observer injection, and nurse-summary output.

## What The Agent Does

```text
Start call
  -> Verify identity
  -> Explain purpose
  -> Check current condition
  -> Screen for red-flag symptoms
  -> Review medications without giving medical advice
  -> Confirm or request follow-up scheduling
  -> Check DME and homecare needs
  -> Ask about caregiver support
  -> Decide next best action
  -> Summarize for the patient
  -> Generate a structured nurse-facing note
```

If the patient reports chest pain, trouble breathing, severe dizziness, confusion, uncontrolled pain, fever, bleeding, worsening symptoms, or similar red flags, the normal intake stops and the agent hands off to an escalation agent.

## Architecture

The call uses three safety and workflow layers:

| Layer | Job | Implementation |
| --- | --- | --- |
| Workflow | Keep the normal call organized and low-latency | `DischargeCoordinatorAgent` + ordered intake tasks |
| Hard interrupt | Stop the checklist when red flags appear | `SafetyScreenTask` -> `EscalationAgent` handoff |
| Continuous guardrails | Catch unsafe requests anywhere in the call | `SafetyPolicyObserver` injects `[POLICY: ...]` hints |

```text
SafetyPolicyObserver
  listens to every user turn
  evaluates recent transcript in the background
  injects policy hints into the active agent

DischargeCoordinatorAgent
  runs identity + safety checks
  runs medication, follow-up, DME, and caregiver tasks
  closes normal calls and generates the nurse summary

EscalationAgent
  takes over on red flags
  provides urgent-care guidance when appropriate
  calls nurse escalation and summary tools
```

## Core Components

| Component | Purpose |
| --- | --- |
| `src/agent.py` | LiveKit entrypoint and voice session setup |
| `DischargeCoordinatorAgent` | Main supervisor for the post-discharge call |
| `EscalationAgent` | Dedicated urgent path for red flags and clinical escalation |
| `SafetyPolicyObserver` | Background guardrail monitor for red flags, medication advice, diagnosis attempts, and worsening condition |
| `VerifyIdentityTask` | Confirms patient name and date of birth |
| `SafetyScreenTask` | Screens for red-flag symptoms |
| `MedicationReviewTask` | Reviews discharge medication instructions while refusing dosage changes |
| `FollowUpTask` | Confirms or requests PCP/specialist follow-up |
| `DmeHomecareTask` | Captures missing walker, oxygen, wound care, home nursing, PT, and similar needs |
| `CaregiverSupportTask` | Records support at home |
| `generate_nurse_summary` | Builds structured JSON from accumulated call state |

## Example Nurse Summary

```json
{
  "patient_id": "P123",
  "call_status": "completed",
  "red_flags": ["shortness_of_breath"],
  "medication_issues": ["patient has not picked up antibiotic"],
  "follow_up_status": "not_scheduled",
  "dme_issues": ["walker not delivered"],
  "caregiver_support": "lives with spouse",
  "next_best_action": "escalate_to_nurse",
  "summary": "Patient reports shortness of breath and has not received walker. Escalated to nurse."
}
```

## Demo Scenarios

Try these in console mode:

- Happy path: confirm identity, no symptoms, meds understood, appointment scheduled, no equipment needs.
- Red flag: "I've been having chest pain since I got home."
- Medication guardrail: "Should I stop taking my antibiotic?"
- DME issue: "My walker never arrived."
- Partial intake: decline a section or say you need to leave early.

## Quickstart

Install dependencies:

```console
uv sync
```

Create a local environment file:

```console
cp .env.example .env.local
```

Fill in your LiveKit Cloud credentials:

```dotenv
LIVEKIT_URL=wss://your-project.livekit.cloud
LIVEKIT_API_KEY=your-api-key
LIVEKIT_API_SECRET=your-api-secret
```

You can also populate `.env.local` with the LiveKit CLI:

```console
lk cloud auth
lk app env -w -d .env.local
```

Download runtime model files before the first run:

```console
uv run python src/agent.py download-files
```

Run an interactive voice-agent console:

```console
uv run python src/agent.py console
```

Run the worker for LiveKit rooms, frontends, or telephony integrations:

```console
uv run python src/agent.py dev
```

Production worker mode:

```console
uv run python src/agent.py start
```

## Patient Metadata

The agent can read patient context from LiveKit job dispatch metadata or room metadata. Metadata should be JSON:

```json
{
  "patient_id": "P123",
  "patient_name": "Jane Doe",
  "expected_dob": "1985-03-15"
}
```

Local development falls back to defaults in `src/discharge/models/call_state.py`.

## Project Layout

```text
src/
  agent.py                         # LiveKit entrypoint
  discharge/
    agents/                        # Coordinator and escalation agents
    models/                        # Typed CallState
    observers/                     # Background safety policy observer
    prompts/                       # Shared voice rules
    tasks/                         # AgentTask workflow steps
    tools/                         # Mock scheduling, DME, escalation, summary tools
tests/                             # Agent behavior, tool, observer, and summary tests
Dockerfile                         # Deployment image
AGENTS.md                          # Project conventions for coding agents
```

## Tests And Quality

Run the full test suite:

```console
NUM_CPUS=2 uv run pytest
```

`NUM_CPUS=2` avoids a macOS sandbox CPU-count issue seen in some local environments.

The behavior tests use LiveKit's agent testing framework and LiveKit Inference. They require valid LiveKit credentials and available LLM token quota.

Focused suites:

```console
NUM_CPUS=2 uv run pytest tests/test_safety_observer.py
NUM_CPUS=2 uv run pytest tests/test_safety_escalation.py
NUM_CPUS=2 uv run pytest tests/test_medication_guardrails.py
NUM_CPUS=2 uv run pytest tests/test_intake_flow.py
NUM_CPUS=2 uv run pytest tests/test_nurse_summary.py
```

Format and lint:

```console
uv run ruff format
uv run ruff check
```


## Deployment Notes

The repository includes a `Dockerfile` suitable for LiveKit agent deployment. For GitHub Actions or any CI environment, configure these as repository secrets:

- `LIVEKIT_URL`
- `LIVEKIT_API_KEY`
- `LIVEKIT_API_SECRET`

Do not commit `.env` or `.env.local`.

## License

This project is licensed under the MIT License. See `LICENSE` for details.
