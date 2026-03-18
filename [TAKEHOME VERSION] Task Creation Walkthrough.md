# Takehome: Improving an AI Agent's Pass Rate on a Hardware Design Task

## Background

 We build evaluation tasks for AI coding agents. An AI agent is given a hardware design problem — a specification and a skeleton of SystemVerilog code — and must write a complete, functionally correct implementation. The agent's solution is then graded by a hidden testbench that checks for bit-exact correctness. It is like an exam problem for a student, where the student is given a problem statement and a skeleton code, and must write the complete solution. The professor has an automated grading system to grade the student's solution.

Your job in this takehome is to **analyze why an AI agent is failing a specific task, then modify the specification to make the problem easier, thereby improving its pass rate**.

You are given a task where an AI agent (Claude Sonnet 4) achieves approximately **10% pass rate** across 20 independent attempts. Your goal is to modify the prompt/specification so that the agent achieves a pass rate **between 40% and 70%** (out of at least 10 runs).

---

## Section 0: Setup (~15 min)

Install the following if you don't already have them:

- [github.com](http://github.com) account
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [Cursor IDE](https://cursor.com) (recommended, but any IDE works)
- [Git](https://git-scm.com/downloads)
- [Python 3.10+](https://www.python.org/downloads/)
- [uv](https://docs.astral.sh/uv/getting-started/installation/) (Python package manager)
- [Icarus Verilog](https://steveicarus.github.io/iverilog/usage/installation.html) (`brew install icarus-verilog` on Mac, `apt install iverilog` on Linux)
- HUD account (you should have received an invite — let us know if not)

**Windows users:** You will need WSL. Follow the instructions in: [WSL Guide](https://docs.google.com/document/d/1LF0nSO5fTD7e_OC6GThE4AWRrnbPHl7Sz-8rnBxjEiM/edit?usp=sharing) (sections 3.1–3.5).

You will also need API keys. Email your liaison to receive:
- An **Anthropic API key** (for calling the AI agent)
- A **GitHub token** with access to the `phinitylabs` org

---

## Section 1: Understanding the Task (~20 min)

### What the agent sees

Clone the task repository:

```bash
git clone https://github.com/phinitylabs/multiply_fp32.git
cd multiply_fp32
```

The task is an **IEEE-754 FP32 floating-point multiplier**. The agent receives:

| File | Purpose |
|------|---------|
| `prompt.txt` | High-level instructions telling the agent what to do |
| `doc/spec.md` | Detailed specification of the multiplier's behavior |
| `sources/multiply_fp32.sv` | Empty module skeleton — the agent must fill this in |

The agent does **not** see the test or the golden solution. It reads the prompt and spec, writes SystemVerilog code, and can run its own tests. After it finishes, a hidden testbench grades its implementation by running 100 random FP32 multiplications and checking for bit-exact correctness against Python's IEEE-754 math.

### Branch structure

The repo has three important branches:

| Branch | What's on it |
|--------|-------------|
| `multiply_fp32_detailed_baseline` | The starting point: empty skeleton + full spec. **This is what the agent sees.** |
| `multiply_fp32_detailed_test` | Same as baseline, plus the hidden cocotb testbench (`tests/test_multiply_fp32.py`). Used for grading. |
| `multiply_fp32_detailed_golden` | Same as baseline, but with the complete, correct implementation in `sources/multiply_fp32.sv`. |

To look at the golden solution:

```bash
git checkout multiply_fp32_detailed_golden
```

To look at the hidden test:

```bash
git checkout multiply_fp32_detailed_test
cat tests/test_multiply_fp32.py
```

To return to the baseline (what the agent starts with):

```bash
git checkout multiply_fp32_detailed_baseline
```

Take time to read `doc/spec.md`, `sources/multiply_fp32.sv`, and the golden solution. Understand what the multiplier does and how the 7-stage FSM pipeline works.

---

## Section 2: Running Tests Locally (~10 min)

You can run the hidden testbench locally to verify that the golden solution passes and the baseline fails.

First, checkout the test branch (which has the testbench):

```bash
git checkout multiply_fp32_detailed_test
```

Run the test against the **baseline** code (should fail — the module has no internal logic, so the testbench will timeout waiting for a response):

```bash
cd tests
uv run pytest test_multiply_fp32.py --log-cli-level=INFO
```

You should see a failure. Now, temporarily swap in the golden solution and re-run:

```bash
cp ../golden/multiply_fp32.sv ../sources/multiply_fp32.sv
rm -rf sim_build __pycache__
uv run pytest test_multiply_fp32.py --log-cli-level=INFO
```

This should pass. Restore the baseline before continuing:

```bash
git checkout -- ../sources/multiply_fp32.sv
cd ..
```

---

## Section 3: Analyzing Agent Behavior (1 hour)

Here are the results of 20 runs of Claude Sonnet 4 attempting this task with the current spec:

**[View the 20 runs here](https://hud.ai/jobs/09836173-f7f4-4cd5-9a9f-8247caea5d20)**

The agent achieves approximately **10% pass rate** — it solves the task correctly in very few of the 20 attempts.

Open 1 or more of the **failing** runs and carefully read through the agent's work. Pay attention to:

- What approach does the agent take?
- Where does it go wrong?
- What does it get right vs. wrong?
- Does it test its own code? What does it miss?
- Are there patterns across multiple failures?

Understanding *why* the agent fails is the key to this takehome. The agent is not randomly broken — there are specific, identifiable reasons it produces incorrect implementations. Your analysis of these reasons will directly inform how you modify the spec.

---

## Section 4: Modifying the Specification (~1–2 hrs)

Your goal is to modify `doc/spec.md` (and/or `prompt.txt`) so that the agent's pass rate improves to **between 40% and 70%**.

The spec currently provides detailed guidance for each pipeline stage. You may modify any part of it — add detail, remove detail, restructure it, add hints, change wording — whatever you believe will help the agent produce correct implementations more often, based on your analysis in Section 3.

A few principles to keep in mind:

- The agent is an AI model, not a human engineer. What helps a human read a spec may not help (or may even hurt) an agent.
- The agent can look things up, write its own tests, and iterate. Consider what information it truly needs vs. what it can figure out on its own.
- Don't give the agent the solution directly (e.g., don't paste the golden code into the spec). The goal is to guide it, not hand it the answer.

Each time you make a change, you should test it by running on HUD (Section 5).

---

## Section 5: Running on HUD (~30 min)

### One-time framework setup

Clone the evaluation framework:

```bash
git clone https://github.com/phinitylabs/verilog-eval-benchmark.git
cd verilog-eval-benchmark
uv sync
```

Set your API keys:

```bash
hud set HUD_API_KEY=<your HUD key from hud.ai → Phinity Labs → API Keys>
hud set ANTHROPIC_API_KEY=<your Anthropic key from liaison>
export GITHUB_TOKEN=<your GitHub token>
```

The problem `multiply_fp32_detailed` is already registered in the framework. After you modify the spec, you need to update the branches, rebuild the Docker image, and run.

### After each spec modification

**1. Update the branches.** From your task repo (`multiply_fp32`), commit your spec change and push to both the baseline and test branches:

```bash
cd /path/to/multiply_fp32
git checkout multiply_fp32_detailed_baseline
# (make your spec edits to doc/spec.md)
git add doc/spec.md
git commit -m "modify spec"
git push origin multiply_fp32_detailed_baseline

git checkout multiply_fp32_detailed_test
git cherry-pick <commit-hash-from-above>
git push origin multiply_fp32_detailed_test
```

**2. Rebuild the Docker image.** From the eval framework repo:

```bash
cd /path/to/verilog-eval-benchmark
docker build --no-cache --platform linux/amd64 \
  -t verilog_multiply_fp32_detailed \
  --build-arg PROBLEM_REPO=multiply_fp32 \
  --build-arg BASELINE_BRANCH=multiply_fp32_detailed_baseline \
  --build-arg TEST_BRANCH=multiply_fp32_detailed_test \
  --build-arg GOLDEN_BRANCH=multiply_fp32_detailed_golden \
  --build-arg PROBLEM_ID=multiply_fp32_detailed \
  --build-arg CACHEBUST=$(date +%s) \
  --build-arg CACHEBUST_GRADING=$(date +%s) \
  --build-arg USE_LOCAL_GRADING=1 \
  --build-arg HINTS=none \
  --secret id=github_token,env=GITHUB_TOKEN .
```

**3. Validate** (confirm golden passes, baseline fails):

```bash
uv run utils/imagectl3.py verilog_ -v --ids multiply_fp32_detailed
```

**4. Push the image and run** (to run 10 attempts):

```bash
docker tag verilog_multiply_fp32_detailed <your-dockerhub-username>/verilog_multiply_fp32_detailed
docker push <your-dockerhub-username>/verilog_multiply_fp32_detailed
```

Update `remote-claude-hud.json` to point to your pushed image (the `Mcp-Image` field), then:

```bash
uv run hud eval remote-claude-hud.json claude \
  --model claude-sonnet-4-20250514 \
  --max-steps 100 \
  --full --remote --group-size 10 -y
```

This will give you a HUD link to view the results. Iterate until you achieve 40–70% pass rate.

---

## Section 6: Submission

Once you have achieved a pass rate between 40% and 70%, email your liaison with:

1. **Your modified `doc/spec.md`** (or a link to your branch)

2. **Your HUD results link** showing the pass rate

3. **A written analysis** containing:

   **A. Root Cause Analysis:**
   Pick 1 or more failing runs from the [original 20-run evaluation](https://hud.ai/jobs/09836173-f7f4-4cd5-9a9f-8247caea5d20). For each, describe:
   - What the agent did wrong (the specific bug in its implementation)
   - Why it went wrong (what caused the agent to make that mistake)

   **B. Faulty Assumptions / Missed Insights:**
   What flawed reasoning, misconceptions, or missing domain knowledge led the agent to produce incorrect code? (e.g., did it misunderstand a concept in the spec? Did it use a Verilog pattern incorrectly? Did it skip a step?)

   **C. Prompt Modifications:**
   What did you change in the spec and why? Connect your changes to your analysis — explain how each modification addresses a specific failure mode you observed in the agent's behavior.

---

This takehome should take approximately **4–6 hours total**. The most important part is the analysis — we want to see that you can diagnose agent failures and reason about how to guide an AI model to produce correct hardware implementations.
