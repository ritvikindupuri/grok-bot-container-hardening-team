# Grok Bot Container Hardening Team

A team of six Grok Bot agents that checks your GitHub repositories for container and Kubernetes security problems and opens pull requests to fix the serious ones.

## Project description

Container setups tend to pick up the same risky mistakes: images that run as root, passwords copied into images, containers given access to the host, Kubernetes workloads with no security settings, and base images that nobody pinned or verified. Checking for all of that by hand across many repositories is slow, and it's easy to miss something.

This project splits that job across a small team of Grok Bot agents. One lead agent, the **Hardening Commander**, takes your request, divides the work, and decides what gets fixed. Five specialist agents each look for one kind of risk. They talk to each other in a shared Grok Bot group chat. Every finding is reported the same way (what the problem is, why it matters, how severe it is, and the exact fix). Serious problems are fixed through draft pull requests that you review before anything is merged.

The team only works defensively. It reads your code and proposes fixes. It never attacks live systems, and it never pastes real secret values into chat.

## Key features

- **Six agents with clear roles.** A team lead plus specialists for images, Kubernetes, secrets, runtime settings, and the software supply chain.
- **Whole-account scanning.** The team finds every repository in your GitHub account that has Dockerfiles, Docker Compose files, Kubernetes manifests, Helm charts, or CI jobs that build images.
- **One consistent report format.** Each finding lists the vulnerability, the risk, the severity with a reason, and the exact fix.
- **Fixes as pull requests.** High and Critical findings are fixed by Cursor cloud agents, which open draft pull requests on the affected repository. Medium and lower findings are reported only.
- **You stay in control.** Nothing is merged without your review.
- **Lab-safe.** Repositories you mark as intentionally vulnerable (for example, security training labs) are scanned and reported, but never changed.
- **Pause and resume.** Tell the group chat to stand down and every agent stops until you say resume.

## Safeguards

These rules keep the team safe to run against real repositories. Each one is also described where it applies elsewhere in this README.

- **Defensive only.** The team reads code and configuration and proposes fixes. It never attacks live systems or clusters and never writes exploit code.
- **Read-only while scanning.** Specialists only read files while they check your repositories. Nothing changes until a fix is opened as a pull request ([How it works](#how-it-works-step-by-step), step 3).
- **Secrets stay redacted.** Real passwords, keys, and tokens are never pasted into chat, findings, or pull requests.
- **Fixes only for serious findings.** Only High and Critical findings get fix pull requests. Medium and lower findings are reported only.
- **Draft pull requests, human merge.** Every fix arrives as a draft pull request on its own branch, and nothing is merged without your review.
- **Lab repositories are never changed.** Repositories you mark as intentionally vulnerable labs are scanned and reported, but never get pull requests ([setup step 3](#3-set-up-the-hardening-commander-and-the-group-chat)).
- **Pause at any time.** Telling the team to stand down stops every agent from starting new work or pull requests until you say resume ([How to use it](#how-to-use-it), step 6).
- **One consistent report format.** Every finding states the vulnerability, the risk, the severity with a reason, and the exact fix, so you can check the reasoning before accepting a change.
- **Grok Bot approval checks.** Grok Bot's built-in approval check can stop risky actions, such as pushing code, and ask you to approve them first. This is a Grok Bot platform feature, not a team rule, and it's controlled in your Grok Bot settings.

## Architecture

<p align="center">
  <img src="docs/architecture.png" alt="How the container hardening team works" width="850">
</p>
<p align="center"><b>Figure 1. How the container hardening team works</b></p>

### How it works, step by step

1. **You ask for a security check.** You message the Hardening Commander (in its own chat or in the team group chat) with what to check, for example "scan all my repos" or "check the Kubernetes files in my API repo."
2. **The Hardening Commander splits up the work.** It finds the repositories that contain container or Kubernetes files and assigns each specialist its part in the group chat.
3. **Each specialist checks your GitHub repositories for one kind of risk.** They read files through Grok Bot's GitHub connection and don't change anything at this stage.
   - **Image Hardener** checks Dockerfiles and images: base images, pinning by digest, running as a non-root user, multi-stage builds, package hygiene (pinned package versions, cleaned caches, no extra packages), health checks, and `.dockerignore`.
   - **Kubernetes Hardener** checks manifests, Helm, and kustomize: security contexts, RBAC permissions, network policies, privileged pods, host access, resource limits, and ingress.
   - **Secrets Hunter** looks for passwords, keys, and tokens left in code, environment variables, build arguments, images, or mounts, plus leaks through missing `.dockerignore` or `.gitignore` files. Values are always redacted.
   - **Runtime Guard** looks for containers running with too much power: privileged mode, dangerous Linux capabilities, a mounted Docker socket, missing seccomp or AppArmor, and shared host namespaces.
   - **Supply Chain Auditor** checks where images and their dependencies come from: unpinned tags such as `:latest`, unpinned build-time downloads (such as `curl | sh`, `npm install` without a lockfile, or GitHub Actions not pinned to a commit), images that are never scanned, missing signing, missing SBOMs (software bills of materials) or provenance, and risky CI build-and-push steps.
4. **Findings come back to the Hardening Commander.** Each one is posted in the group chat as *Vulnerability | Risk | Severity (and why) | Exact fix*.
5. **Serious problems get a fix.** For each High or Critical finding, the Hardening Commander (or the specialist it assigns) launches a Cursor cloud agent. The cloud agent creates a branch, applies the fix, and opens a draft pull request. Medium and lower findings are only reported.
6. **You review the fix.** You read the pull request and decide whether to merge it.

### How the team coordinates and settles conflicts

Five specialists working on the same repositories will sometimes overlap or disagree. The Hardening Commander handles that in the group chat, where every decision and the reason for it stays visible. These are the patterns it followed in a real run. They come from the Commander's judgment as team lead, not from code that enforces them.

- **One owner per file area.** When two specialists need to change the same lines, for example both want to edit the same `FROM` line in a Dockerfile, the Commander assigns the change to one pull request. The other specialist hands over its part (such as an image digest) instead of opening a second pull request that would conflict.
- **Peer review before a fix is marked ready.** When a specialist has nothing to fix in a repository, the Commander can assign it to review the others' fixes. If the review finds a gap, such as a fix that still leaves a dangerous path open through another route, the owning specialist fixes it on the same pull request before the Commander marks it ready.
- **Conflicting recommendations get a decision with reasons.** When specialists recommend fixes that don't work together, for example one change would break a protection another pull request already adds, each states its case in the group chat. The Commander weighs the risk of each option, picks one, records why, and moves the rejected option to the follow-up list if it's still worth revisiting.
- **Overlapping pull requests get a merge order.** When two fix pull requests touch the same part of a file, the Commander sets a merge order and test-merges them together so the order works without conflicts.
- **Scope stays fixed.** Only High and Critical findings get pull requests. Medium and lower findings, and any options the Commander rejected, go on a follow-up list in its final summary.

## Tech stack

- **Grok Bot** runs the six agents, their group chat, memory, and messaging between agents.
- **GitHub connection in Grok Bot** lets the agents search and read repositories, pull requests, and CI runs.
- **Cursor cloud agents** make the code changes on a branch and open the pull requests.
- **The files being checked:** Docker and Containerfiles, Docker Compose, Kubernetes manifests, Helm charts, kustomize, and GitHub Actions workflows.
- **Supply chain tools used in the fixes:** `crane` to look up image digests for pinning, plus Trivy (vulnerability scanning), Syft (SBOM generation), and Cosign (image signing) in a GitHub Actions scan gate.
- **Kubernetes controls used in the fixes:** Pod Security Standards, security contexts, NetworkPolicy, RBAC, seccomp, and Kyverno admission policies.

## Set it up in your own Grok Bot

You need a Grok Bot account, a GitHub account, and access to Cursor cloud agents. Setup takes four steps.

### 1. Import the six agents

Each agent is published as a public Grok Bot template. Open each link and import it into your Grok Bot. A template brings the agent's name, role, rules, and finding format with it, so there's nothing to type in by hand.

| Agent | What it does | Template |
|-------|--------------|----------|
| Hardening Commander | Team lead. Splits up the work and decides what gets fixed. | [Import](https://x.ai/bot/JBtMhvtYo2R1aky5s6xXK) |
| Image Hardener | Checks Dockerfiles and container images. | [Import](https://x.ai/bot/441XDiyC7uaji-U66ci-3) |
| Kubernetes Hardener | Checks Kubernetes settings, permissions, and networking. | [Import](https://x.ai/bot/IVmcj7aFV-wtMA19P3G4Q) |
| Secrets Hunter | Finds passwords and keys left in code or images. | [Import](https://x.ai/bot/LkzOnNaqM1tuEYZTFwzG_) |
| Runtime Guard | Finds containers running with too much power. | [Import](https://x.ai/bot/5OSZJ5JKxsiThQ0Nnr-rJ) |
| Supply Chain Auditor | Checks where images come from and whether they're verified. | [Import](https://x.ai/bot/qbxHfvaOU71YTY5-nZSAt) |

### 2. Connect GitHub

Open the Hardening Commander's chat and ask it to "connect my GitHub." (If it starts with its setup questions first, that's fine. Its first question is about GitHub, and it will walk you through connecting.) Approve the connection card that appears, and give Cursor access to the repositories you want checked. The agents use this connection to read code, and the Cursor cloud agents use it to push fix branches and open pull requests, so the repositories you want fixed need write access, not just read access. Also allow access to workflows (GitHub Actions files under `.github/workflows/`). GitHub treats those separately from normal code, and without that permission any fix that adds a CI scan gate is rejected when it's pushed, even though other fixes still go through. You only connect GitHub once; all six agents share the connection.

### 3. Set up the Hardening Commander and the group chat

In the same Hardening Commander chat, answer its first-run setup questions. It asks a few questions one at a time: your GitHub username, whether any of your repos are intentionally vulnerable labs (those are scanned but never changed), and whether it should open fix pull requests for High and Critical findings on its own or check with you first. It saves your answers.

Then ask it to create the team room, using this message:

> Create a group chat named "Container Hardening Command" with Image Hardener, Kubernetes Hardener, Secrets Hunter, Runtime Guard, and Supply Chain Auditor, and give it this description: Hardening Commander calls the shots: scopes repos, assigns specialists, consolidates findings, prioritizes risk, and pushes exact fixes. Image Hardener owns Dockerfiles and images. Kubernetes Hardener owns manifests, RBAC, networking, and pod security. Secrets Hunter finds leaked credentials and bad mounts. Runtime Guard hunts privileged execution, dangerous capabilities, sockets, and namespace abuse. Supply Chain Auditor checks image trust, tags, SBOMs, provenance, and CI risks. Find, decide, remediate.

### 4. Answer each specialist's setup questions

Open each specialist's chat once. Each one asks a short set of questions the first time. They all ask which GitHub account to scan, which repos are find-only labs, and who they report to (some word this as whether they work with a team lead). Some also ask whether they should open fix pull requests, and one may offer to let you watch its work live. Give the same answers you gave the Hardening Commander, and when asked who it reports to or about a team lead, answer with the `Container Hardening Command` group chat. That way every specialist posts its findings where the Hardening Commander can collect them.

Once all six have their answers, the team is ready. Go to [How to use it](#how-to-use-it) to start the first scan.

<details>
<summary><b>Prefer to build the team by hand instead of importing?</b></summary>

Create six agents with these names and descriptions, then connect GitHub (step 2), create the group chat (the message in step 3), and send the standing rules below. Replace `<your-github-username>` with your own username.

**Hardening Commander**

- **Description:**
  > Leads the container and Kubernetes hardening team. Assigns full scans across every repo in my GitHub account that has Dockerfiles, Compose, Kubernetes/Helm, or CI image builds. Consolidates specialist findings, decides what gets fixed, and opens fix pull requests for High and Critical findings through Cursor cloud agents. Commands Image Hardener, Kubernetes Hardener, Secrets Hunter, Runtime Guard, and Supply Chain Auditor. Defensive work only: images, Kubernetes, runtimes, secrets, and supply chain.

| Name | Description |
|------|-------------|
| `Image Hardener` | Searches every `<your-github-username>` repo that has Dockerfiles, Containerfiles, or image builds. Finds weak image settings (base images, digests vs tags, multi-stage builds, non-root USER, package hygiene, HEALTHCHECK, .dockerignore) and gives exact Dockerfile or Compose fixes. No live exploits. Reports to Hardening Commander with severity, path, risk, and the exact fix. |
| `Kubernetes Hardener` | Searches every `<your-github-username>` repo with Kubernetes manifests, Helm, or kustomize. Finds risky defaults (SecurityContext, NetworkPolicy, privileged pods, hostPath/hostNetwork, RBAC, limits, ingress) and gives exact manifest patches. No live cluster attacks. Reports to Hardening Commander with severity, resource path, risk, and the exact fix. |
| `Secrets Hunter` | Searches every `<your-github-username>` repo with containers, Compose, or Kubernetes for credentials in ENV, keys baked into images, plaintext secrets, bad mounts, and leak paths. Never pastes real secret values and always redacts them. Reports to Hardening Commander with severity, location, risk, and the exact fix. |
| `Runtime Guard` | Searches every `<your-github-username>` repo with Compose or runtime configs for privileged mode, dangerous capabilities, missing seccomp or AppArmor, docker.sock mounts, and shared namespaces. Defensive analysis only, no exploit code. Reports to Hardening Commander with severity, location, risk, and the exact lock-down. |
| `Supply Chain Auditor` | Searches every `<your-github-username>` repo that builds or pulls images for unpinned tags, unsigned images, missing SBOMs or provenance, and risky CI build-and-push steps. Reports to Hardening Commander with severity, location, risk, and the exact fix (pin, sign, scan, or add CI gates). |

Then send the Hardening Commander these standing rules and ask it to remember them and share them with the team:

- Every finding uses the format *Vulnerability | Risk | Severity (and why) | Exact fix*.
- Open fix pull requests only for High and Critical findings. Report Medium and lower.
- These repositories are intentionally vulnerable labs. Scan and report them, but never change them: `<list your lab repos, or say "none">`.
- The Hardening Commander decides what gets fixed; specialists work it out among themselves and don't ask me to approve each step.
- Never paste real secret values. Redact them.

</details>

## How to use it

1. **Start a full scan.** In the `Container Hardening Command` group chat, write something like:
   > Scan every repo in my GitHub account that has containers or Kubernetes files, and report findings.
2. **Watch the reports come in.** Each specialist posts its findings in the group chat in the standard format. The Hardening Commander collects them and posts a prioritized summary.
3. **Let the fixes open.** For High and Critical findings, the team launches cloud agents that open draft pull requests. Each pull request description repeats the finding, so you can see why the change was made.
4. **Review and merge.** Open each pull request on GitHub, check the diff and CI results, and merge the ones you want.
5. **Target a single repo or a single area** when you don't need a full scan:
   > Check only the Kubernetes manifests in `my-api-repo`.

   > Secrets Hunter, re-check `my-web-app` after the last merge.
6. **Pause the team** at any time:
   > Take a break and don't do anything until I say resume.

   Every agent stands down and starts no new work or pull requests. GitHub Actions runs that have already started keep running on GitHub. Say "resume" to pick up where you left off.
7. **Ask for status.** Ask the Hardening Commander "what's open?" for a list of open fix pull requests and anything still in progress.

## Sample findings

To see what the team's reports actually look like, read [SAMPLE_FINDINGS.md](SAMPLE_FINDINGS.md). It has real screenshots from the group chat: Runtime Guard's runtime re-check across every repo, and Kubernetes Hardener and Supply Chain Auditor reporting the fix pull requests they opened.

## Limitations

- The team reads code and configuration. It does not scan running clusters or live containers.
- Severity ratings are the agents' judgment. Review each pull request before merging.
- How many fixes can run at once depends on your Cursor cloud agent limit. The Hardening Commander queues work when that limit is reached.
