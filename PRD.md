# Product Requirements Document (PRD): Promptly

Version: 1.0, including review and browser-local history improvements

Document date: 15 November 2026

Status: Documents the implemented application

## 1. Product overview

Promptly is a text workspace for summarizing, rewriting, and extracting action items. A person supplies source text, chooses one of three operations, reviews the structured result, and copies or downloads the version they want to use. The application keeps the source accessible throughout that workflow.

The product is one main screen with a supporting history drawer and guide dialog. It uses React, Express, and a configurable AI provider. It also runs without credentials in an explicitly labeled demo mode.

## 2. Problem and intended users

Students, writers, and people working with team updates often need to understand long passages, improve rough messages, or identify concrete next steps. Reformatting a free-form AI response and checking it against the source adds work. Promptly provides purpose-built output for each operation and makes review part of the flow.

Typical uses include shortening research notes, refining an email, and converting meeting notes into a checklist. These are intended use cases, not claims of measured customer adoption or productivity gains.

## 3. Goals and scope

- Make each operation understandable without learning prompt syntax.
- Return predictable fields that can be rendered, edited, and exported.
- Preserve the generated version and original source during result editing.
- Let people resume successful work on the same browser.
- Keep the application small enough to run locally and explain during a project demonstration.
- Keep provider credentials on the server and make failure states understandable.

The scope excludes accounts, authentication, payments, a server database, file uploads, cloud history, collaborative editing, and multi-device synchronization. Past tasks are browser-local snapshots. The application does not execute extracted tasks or send messages to other people.

## 4. User journeys

### 4.1 Understand a passage

Paste text or choose a sample, select Summarize, choose concise or detailed, and submit. Review the overview, key points, and takeaway. Edit any of those fields if needed, then copy or download the result.

### 4.2 Improve a message

Select Rewrite and a tone, supply text, and submit. Read the revised draft and the engine's explanation of its changes. Show changes to compare the current draft against the source. Personal edits can be reset to the generated draft.

### 4.3 Organize meeting actions

Select Extract tasks and submit meeting notes. Review each task's wording, owner, deadline, priority, and source evidence. Find an exact quote in the original, correct task details, check completed tasks, and optionally hide completed rows. Export includes the full checklist and completion state.

### 4.4 Resume previous work

Open Past tasks, optionally filter by tool, and reopen a saved result. The source, preferences, generated result, edits, and completion are restored without another AI request. Delete individual entries or confirm Clear all to remove local history.

## 5. Functional requirements and acceptance criteria

| ID | Requirement | Acceptance criteria |
| --- | --- | --- |
| FR-01 | Source editor | Text can be typed, pasted, cleared, or replaced with a tool-specific sample. Word and character counts update. Invalid or excessive input receives clear feedback. |
| FR-02 | Summarize | Concise and detailed options produce a result containing an overview, a key-point list, and a takeaway. |
| FR-03 | Rewrite | Professional, casual, concise, and polished tones are selectable. The result contains revised text and a change explanation. |
| FR-04 | Extract tasks | Output contains an overview and zero or more tasks with wording, nullable owner/deadline, explicit or unspecified priority, and supporting evidence. |
| FR-05 | Source review | A nonempty evidence quote can be located only if it exactly occurs in the original. The matching range is highlighted, and editing focus can be restored. |
| FR-06 | Result editing | Summary fields, rewritten text, and task details are editable. Source text and task evidence are not altered by result editing. An edited label distinguishes local changes. |
| FR-07 | Reset edits | Reset restores generated content while retaining task completion. Starting another generation processes the source, not the edited result. |
| FR-08 | Rewrite comparison | Added and removed text is distinguished with text decoration and color. Comparison follows the current edited draft and does not require AI. Large differences may be grouped. |
| FR-09 | Task progress | Checkboxes determine the completion total. Hide completed filters visible rows. An all-complete message appears when the filtered view is empty. |
| FR-10 | Copy and download | Copy and plain-text download use the currently displayed result. Task exports include all tasks, even when completed rows are hidden. Success or failure is reported. |
| FR-11 | Past tasks | Up to 20 successful results are retained within a bounded browser-storage budget, with newest generation dates first. Reopening does not call the provider. |
| FR-12 | History deletion | Individual deletion removes that saved entry. Clear all requires confirmation. An open result remains usable and is not automatically recreated in history after deletion. |
| FR-13 | Processing feedback | Empty, loading, error, success, cancellation, and retry states are available. Late responses cannot replace newer work. |
| FR-14 | Layout and focus | Desktop shows source and output side by side. Smaller screens show Original or Result one pane at a time. Focus view removes the greeting to expand the work area. |
| FR-15 | Keyboard access | Controls have accessible names and visible focus. Ctrl/Cmd+Enter submits from the source editor. Native dialogs support Escape and focus containment. |
| FR-16 | Demo mode | With no credentials, sample fixtures and limited local rules allow a full workflow demonstration. Every demo result is labeled. |
| FR-17 | Real AI mode | Configured Gemini or OpenAI runs only through the backend. Missing credentials in real mode cause a startup error; provider failures never silently switch to demo. |

## 6. Data and validation requirements

Requests use the single `POST /api/process` endpoint. The trimmed source must contain at least 10 characters, no more than 20,000 characters, and at most 4,000 whitespace-separated words. JSON bodies are limited to 100 KB. Invalid modes, preference values, and unknown request keys are rejected.

Generated output must pass the operation's Zod schema before being returned and displayed. Task instructions require unsupported owners/deadlines to remain null and unsupported priorities to be unspecified. Schema compliance does not prove that a summary or task is factually correct; the user must review it.

Only successful processing snapshots and subsequent result edits/completion are retained in Past tasks. Unfinished source-only drafts, failed requests, and credentials are not history entries. Browser data is specific to the site address and can be removed by browser clearing or user deletion.

## 7. Non-functional requirements

| Area | Requirement and present boundary |
| --- | --- |
| Privacy | No source/result database exists on the server. Local history contains readable text. Real AI sends source to the configured provider, whose policies apply. |
| Security | Keep API keys out of browser bundles, URLs, Git, and error messages. Enforce configured origins, body/rate limits, and safe error responses. Production uses same-origin content policies. |
| Reliability | A provider timeout, malformed response, refusal, network error, or storage error has a useful fallback/message. Browser storage failure preserves session work. |
| Responsiveness | The existing work area remains usable at 320, 390, 768, 1024, and 1440px viewport widths. Documents scroll inside their panes. |
| Accessibility | Use semantic controls, labels, focus feedback, source-review focus handling, written state feedback, and reduced-motion support. A complete formal WCAG certification is not claimed. |
| Performance | AI generation has bounded timeouts. Rewrite comparison uses a bounded matrix. No throughput or response-time service-level agreement is claimed. |
| Maintainability | Shared contracts, separate provider adapters, pure workspace/history/comparison logic, tests, and documentation keep concerns explicit. |

## 8. Release acceptance and demonstration

The implemented revision has previously passed 68 automated checks, 25 browser scenarios, and a production build. Details and test limitations are in [TESTING.md](TESTING.md). These results describe the application revision; this documentation addition does not imply a new live provider test.

A release demonstration should show all three tools, at least one invalid request/error recovery, edited output export and reset, task evidence/progress, history restoration after reload, and phone navigation. Demo mode is sufficient for repeatable checks; a real provider demonstration requires the owner's server-side credentials and available quota.

## 9. Known limitations

AI can produce unsupported statements despite valid output structure. The demo is deliberately less capable than real AI. History has no cloud backup, encryption, or multi-tab conflict reconciliation. Canceling a browser request does not guarantee that a provider stops generating or billing. The in-memory rate limiter and lack of authentication do not provide account-level protection for a broadly exposed paid API.

See [HLD.md](HLD.md) for the architecture, [LLD.md](LLD.md) for contracts and algorithms, and [CONCEPTS.md](CONCEPTS.md) for the course concept mapping.
