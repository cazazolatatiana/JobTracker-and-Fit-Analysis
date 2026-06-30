# JobTrack

A personal job search operating system, built end to end in a weekend and refined through daily use since.

JobTrack replaced a fragmented manual workflow, copying job postings into a notes tool, re-explaining context to an AI assistant in a new chat each time, losing track of why a CV decision was made, with a single connected system across a Chrome extension, a Supabase backend, and a dashboard.

This is a private portfolio project, not a published product. It exists to demonstrate how I think about and build process automation, including the parts that went wrong along the way.

<img width="692" height="370" alt="JobTrack extension sidebar" src="https://github.com/user-attachments/assets/a5d1f869-b577-40ee-ba50-ff9744c560a5" /> <img width="691" height="371" alt="JobTrack dashboard pipeline" src="https://github.com/user-attachments/assets/dfff3854-d870-42c0-83a4-1796d90b90a8" />

---

## Why this exists

Job searching across multiple markets generates a lot of repeated work. Every job posting needs reading, scoring against a profile, and turning into a tailored CV. Every networking conversation needs tracking against the application it supports. None of the obvious tools did all of this in one place.

A polished community template for tracking job applications already existed. It could have been the answer. The real problem was that any ready-made tool still meant manual work, copying each job description across by hand, re-entering fit scores and CV reasoning that had been worked out separately in a disconnected AI chat. The capture itself was never automated the way it needed to be. A better notes template would have fixed the surface problem and left the structural one untouched, every part of the process still living in a different place with no shared memory between them.

So instead of adopting a better template, I built the workflow around shared data.

## What it does

**Capture.** Open a job posting in any tab, open the extension, click *Grab from page*. The listing gets read directly from the page, no copying and pasting.

**Score.** The job description goes to a two-stage AI pipeline. A fast model distills it into structured data (seniority, required skills, must-haves, gaps). A stronger model scores it against a stored profile and explains why.

**Decide.** Mark the job *Interested* or *Not for me* from the same sidebar. Interested kicks off CV recommendation generation in the background. Not for me archives it but keeps it reachable.

**Manage.** The dashboard turns every captured job into a pipeline view, status, fit score, CV recommendations, follow-up dates, all filterable and searchable, with a single combined feed of everything due today across both applications and networking outreach.

**Track networking.** Contacts live independently of any one job, but every outreach action can be linked to the application it supports, so "did I ever follow up with this person" stops being a guess.

## How it's built

```
Chrome extension (capture)  ──┐
                               ├──►  Supabase  ──►  Claude API
Dashboard (manage)         ───┘      (Postgres,        (Haiku + Sonnet)
                                       Edge Functions,
                                       Storage)
```

Both the extension and the dashboard are independent applications sharing one Supabase backend. A job captured from the sidebar appears in the dashboard immediately, since both read and write the same tables directly, no custom backend layer in between.

The AI work runs in two stages for cost reasons. Claude Haiku handles job description distillation, a mechanical extraction task that doesn't need a frontier model. Claude Sonnet handles fit scoring and CV recommendation generation, where reasoning quality actually matters. Routing the easy part to the cheap model meaningfully cuts the cost of the expensive call that follows.

Most of the dashboard's interface was built by directing Lovable, a low-code AI builder, through detailed natural-language specifications, applying existing knowledge of SQL and system design to specify exactly what each screen and data relationship needed to do, rather than writing every line by hand.

## What broke, and what that taught me

Documenting these honestly, because the debugging process is part of the point of this project, not despite it.

A decision was saving in the UI but never reaching the database. The flag that gated the actual save had been left tied to a screen that no longer existed after a UX simplification, so the interface looked correct while quietly writing nothing.

CV recommendations stopped generating with zero server-side logs, which made it look like the request never arrived. It did arrive. The response schema had grown past the token limit set on the AI call, the model's output was getting cut off mid-JSON, and the parser was failing several layers downstream of where the real cause was.

A storage bucket for CV uploads returned a permissions error after being created correctly, because Supabase Storage has its own access policy layer separate from the database's, and the first attempt to write that policy through the UI silently got wrapped in a template that broke it.

None of these were exotic problems. They were the ordinary kind that show up once a system is actually used rather than just demoed, and the value was in tracing each one to its real cause instead of patching the symptom.

## Project structure and technical detail

This repository is split into two parts, each with its own full technical README.

**[JobTrack Extension](../job-tracker-extension)** — the Chrome sidebar that captures job postings and logs decisions. Built with Plasmo. Covers the page-scraping strategy per job board, the Edge Functions it calls, and known limitations.

**[JobTrack Dashboard](../job-tracker-dashboard)** — the pipeline management interface. Built with TanStack Start, React, and Supabase. Covers the full database schema, the two-stage AI pipeline in detail, and setup instructions.

## Stack

| Layer | Tools |
|---|---|
| Capture | Chrome extension, Plasmo, React |
| Backend | Supabase (Postgres, Edge Functions, Storage) |
| AI | Claude API, Haiku for distillation, Sonnet for scoring and recommendations |
| Dashboard | TanStack Start, TanStack Query, shadcn/ui, Tailwind |
| Build tooling | Lovable, used for most of the dashboard's interface |
