# I Set Out to Buy a Second GPU. I Ended Up Building an Autonomous Coding Loop.

A day that started with a simple hardware question ended with an AI agent building a Tetris clone on my own hardware, running on its own. Almost every step in between was me holding a confident assumption, testing it, and watching it fall apart. This post is the log of those corrections, because the corrections are the useful part.

The setup: a desktop with a Radeon RX 7800 XT (16GB), a Ryzen 7 5800X3D, and 48GB of RAM, plus a MacBook Air. I used an AI assistant throughout to research options and pressure-test decisions. It was a strong partner, and it was also confidently wrong at one critical moment, which became one of the day's best lessons.

If there is a single takeaway, it is this: measure, do not assume. Nearly every belief I started a question with was wrong, and a log or a quick test corrected it every time.

## Would a second GPU make this faster?

That was the original question. I wanted to run a local model for a coding agent and figured 16GB was the constraint, so a second identical card to reach 32GB seemed like the obvious upgrade.

The first correction came fast. A second consumer GPU buys you *capacity*, not *speed*. Tools like llama.cpp split a model across cards by layers and run them in sequence, so two cards do not double throughput on a model that already fits on one. They can even be slightly slower from the cross-card handoff. Token generation is bound by memory bandwidth, and layer splitting does not raise the effective bandwidth for a single request.

The only time a second card makes things faster is when it lets a model that previously spilled into system RAM fit entirely in VRAM. That is a capacity win wearing a speed costume. So the real question was never "more VRAM," it was "which model, and does it fit."

## Docker was the right instinct and the wrong tool

I wanted clean, reproducible deployment. With a Dockerfile and a vanilla base image, you know exactly what is installed and can trust the setup is not reaching for something unexpected. That instinct is correct almost everywhere.

It inverts on Windows with an AMD card. Docker Desktop runs containers inside a WSL2 virtual machine, so getting the GPU into a container means tunneling through ROCm-on-WSL and, in practice, punching holes in the very isolation that made Docker appealing. The Dockerfile stops being the whole story, because a chunk of what is actually installed now lives in a WSL layer the file does not capture. You end up less certain of your setup, not more.

The lesson: the right default tool can be the wrong tool in a specific environment. On Windows with AMD, native is the legible path. I wrote down the dependency surface by hand instead, and it was small: a GPU driver, a self-contained llama.cpp build, a model file, and a launch command.

## The model rabbit hole, and the trap at the bottom of it

This was the heart of the day.

I wanted a capable model for agentic coding. The first lever was architecture. A Mixture-of-Experts model with roughly 3B active parameters per token behaves very differently from a dense model on a 16GB card, because you can keep the attention and routing layers on the GPU and offload the bulky expert weights to system RAM. Only a fraction of the weights fire on any given token, so that offload hurts far less than it would for a dense model.

I tuned it the slow, correct way: load the model, watch VRAM, and walk the "force experts to CPU" count down from every layer to fewer layers until dedicated memory sat at about 15.3 out of 16GB with no spillover. That tuning loop was satisfying and it worked.

Then I hit the trap. The model I was running used a hybrid linear-attention design, and that design cannot cache prompts. In an agent loop, where most of each turn's prompt is identical to the last, that is brutal. Every single turn re-processed the entire prompt from scratch. I watched the logs show roughly 28,000 tokens re-ingested each turn, which was about 70 seconds before the model produced a single new token. No amount of GPU tuning fixes that, because it is a property of the architecture, not the hardware.

Here is the honest part. My AI assistant confidently recommended switching to the "dense" 27B version of the same family as the fix. It sounded reasonable. Before downloading 17GB, I asked it to verify the architecture, and the recommendation fell apart: "dense" only means no expert routing. It does not change the attention. The 27B used the exact same no-cache design and would not have helped at all. The model was wrong with conviction, and the only thing that caught it was checking the claim against the actual model card before acting.

That is worth sitting with. The tool that is accelerating your work will sometimes be precisely, fluently wrong. Verification is not optional, especially right before an irreversible step.

The real fix was a model that uses ordinary attention, which caches normally. I weighed a coding-specialist option that caches cleanly but is text only, and landed on a small multimodal model that gave me caching, image input, and a tiny footprint at once. The proof was in the log: a follow-up turn on a 24,000-token conversation re-processed in about 2 seconds, because only the new tokens were prefilled and the rest came from cache. That single log line was the whole point of the switch.

## I spent two rounds trimming the wrong thing

With the model sorted, the agent harness was slow and kept overflowing the context window. The visible culprit was tools: the agent had 43 of them loaded, each injecting a schema into every request. I trimmed them to 29. The prompt size barely moved.

The data was telling me something I did not want to hear. The bulk of the prompt was not the tool schemas, it was the agent framework's base system prompt, which is large and not something you can trim from the UI. I had been optimizing the visible knob instead of the load-bearing one. The fix was not fewer tools, it was a leaner harness entirely. On a small-context local model, the size of the harness prompt matters more than almost anything else, and the heavyweight in-editor agents are built for frontier cloud models, not for this.

## The Ralph loop, and the skills that feed it

Everything came together around a technique called the Ralph Wiggum loop, named and shared by Geoffrey Huntley in mid-2025. The idea is almost dumb in its simplicity: run an agent in a shell loop, give it a fresh context window every single iteration, and let it read its plan from disk, do one task, and exit. State lives in git history and a plan file, not in the model's context. Completion lives outside the model, in test commands that exit zero or non-zero. The model never decides it is done; the harness does.

This is a perfect fit for a small-context model, because you never accumulate context. The plan file is the memory.

I built two reusable skills to support it:

- **ralph-plan** takes any loose plan and restructures it into a loop-ready format: small checkbox tasks, one objective acceptance command per task, sensitive tasks flagged for human review, and a clear definition of done. The judgment it encodes is the valuable part. Every task needs a check a machine can run, or it gets marked as a human call. Vague tasks get bounced back rather than papered over.
- **ralph-init** scaffolds a folder into a runnable loop: it emits the loop script, the pinned prompt, and the provider config, wires up git, and adds the stop conditions.

There was one more insight worth its own paragraph. My first pass marked all the rendering and UI work as "manual," because canvas output is not something a unit test can read. That was lazy. With a browser test runner and a small test seam (the app publishes its live state and a few deterministic hooks on a global object), most of that becomes objectively testable. Does the piece move on a keypress? Does clearing a line raise the score? Does the layout survive a small viewport? All of that is automatable. The rule became: test behavior with end-to-end checks, reserve "manual" only for genuine taste, like whether the animation feels good. That pushed my truly-manual task count from nine down to three.

## Autonomy does not remove the human, it relocates the leverage

The thing I kept circling back to: handing the loop full autonomy does not delete the human gate, it moves it. With no person approving each step, two places become more important, not less.

The first is the spec. If nobody is steering mid-run, the plan is the program, and a vague plan produces a confident mess. The unhurried review goes here, before launch.

The second is risk. Anything touching credentials, data, money, or irreversible actions gets flagged to halt and wait for a human. Huntley's phrasing stuck with me: your job is to sit on the loop, not in it.

## The last bug was the best one

The first real run failed immediately. The agent tried to reach Google's hosted API and reported no API key, instead of talking to my local server. The cause was a parsing assumption: given a model id with a slash in it, the agent split on the slash and treated the first part as a cloud provider. The local server was never contacted. The fix was to register the local server explicitly as a custom provider and call the agent with that provider by name.

But look at what the loop did with the failure. It caught the non-zero exit, refused to commit nothing, and a "stuck on the same task" guard stopped the run after exactly three dead iterations instead of spinning forever. The guardrails I had built proved themselves on a real failure before I had even gotten a successful run. That is the kind of thing you only trust after you watch it happen.

I added a preflight check so that class of misconfiguration now fails in two seconds with a precise pointer, rather than burning iterations. Then the loop ran.

## What I actually proved, and what I did not

I want to be precise about the win. I proved the pipeline: a model that fits, caches, and runs fast on my hardware; a plan in a format an autonomous loop can execute; a harness that drives tasks, commits each step, and stops at the right human boundaries. I did not prove the model writes good Tetris. That is the next thing to watch, and it is a separate question from whether the machinery works.

The throughline was never Tetris, or even the loop. It was the discipline of distrusting my own confident beliefs. More VRAM equals faster. Docker equals cleaner. Dense equals caching. Fewer tools equals smaller prompt. Every one of those was wrong, and a log, a dry run, or a quick verification corrected each one. That habit is the transferable skill. The specific stack will be obsolete in a year. Measuring before you commit will not be.
