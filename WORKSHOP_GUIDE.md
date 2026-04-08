# Workshop Guide: Self-Healing AIOps Demo

A presenter companion for `Workshop-SelfHealing-Local.ipynb`.

---

## Prerequisites

- Python 3.12 installed
- The `.venv/` directory is already set up with all dependencies (jupyter, flask, google-generativeai)
- The Gemini API key is already embedded in the Setup cell

## Launching the Notebook

Open a terminal in the project directory and run:

    cd /home/imran/Documents/personal/noida
    .venv/bin/jupyter notebook Workshop-SelfHealing-Local.ipynb

Your browser will open. Run cells **top to bottom** using Shift+Enter.

---

## End-to-End Run Guide

| Section | Cell to run | Expected output |
|---------|------------|-----------------|
| Setup | Setup code cell | `Setup complete. Gemini Flash initialized.` |
| Stage 1 Deploy | Deploy code cell | `Server health check: HEALTHY ✓` |
| Stage 1 Attack | Slowloris code cell | `UNRESPONSIVE ✓ (attack working)` |
| Stage 1 Diagnose | Diagnose code cell | Gemini's diagnosis + proposed fix printed |
| Stage 1 Apply | Apply code cell | `HEALED ✓` |
| Stage 2 Attack | Thread exhaustion code cell | `UNRESPONSIVE ✓ (thread pool exhausted)` |
| Stage 2 Diagnose | Diagnose code cell | Gemini's new diagnosis + proposed fix |
| Stage 2 Apply | Apply code cell | `HEALED ✓` |
| Cleanup | Cleanup cell | `Lab cleanup complete.` |

---

## How the Attacks Work

### Stage 1: Slowloris

Slowloris is a denial-of-service technique that uses very little traffic to take down a server. Instead of flooding it with requests, it opens many connections and keeps them alive by sending incomplete HTTP headers — never finishing the request.

With `threaded=False`, Flask uses a single thread for all requests. Slowloris occupies that one thread permanently. No other client can be served. 150 connections are opened to make the effect immediate.

**Why it stops working after the fix:** `threaded=True` tells Flask to spawn a new thread per incoming request. Slowloris connections still hold threads — but the OS can spawn thousands of threads. 150 is no longer enough to saturate the pool.

### Stage 2: Thread Exhaustion

Once the server is multi-threaded, Slowloris is ineffective. But the `/data` route calls `time.sleep(10)`, which blocks the thread for 10 full seconds. We fire 20 concurrent requests to `/data`. Each one immediately occupies a thread that does nothing but sleep. Within 3 seconds, all available threads are busy. New requests (including the health check) queue indefinitely.

**Why it stops working after the fix:** Gemini removes or reduces the sleep, so threads are released quickly. Concurrent requests no longer pile up.

---

## How the Self-Healing Agent Works

Each stage runs the same two-step loop:

**Step 1 — Diagnose (Cell A):**
1. Reads the current `app.py` from disk
2. Builds a prompt containing: the app source code, a description of the incident, and the observed symptoms
3. Sends the prompt to Gemini Flash via the `google-generativeai` SDK
4. Parses Gemini's response — splits on `DIAGNOSIS:` and `FIXED CODE:` markers, extracts the fenced Python block
5. Prints the diagnosis for the audience and stores the fixed code in `healed_code`

**Step 2 — Apply (Cell B):**
1. Runs `compile()` on `healed_code` — refuses to write invalid Python to disk
2. Shows a diff of what changed (lines removed and added)
3. Overwrites `app.py` with the fixed code
4. Terminates the running server and starts a fresh one
5. Runs a health check and reports `HEALED ✓` or `STILL DOWN ✗`

The agent adapts: in Stage 2 it reads the already-patched code and targets only the new vulnerability, not the one it already fixed.

---

## Common Failures and Recovery

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `Server did not start` after Deploy | Port 5000 in use from a previous run | Run the Cleanup cell, then re-run Deploy |
| Stage 1 attack shows `HEALTHY` | Server started too fast before attack | Re-run the attack cell |
| `WARNING: Could not parse fixed code` | Gemini responded in an unexpected format | Re-run the Diagnose cell — responses vary slightly |
| `Syntax error in Gemini's fix` | Gemini returned broken Python | Re-run the Diagnose cell |
| Stage 2 attack shows `HEALTHY` | Thread pool not exhausted yet | Increase `num_requests` to 50 in the attack cell |
| Kernel dies mid-demo | Memory issue | Restart kernel, re-run from Setup cell |

---

## Resetting Mid-Demo

If something goes wrong partway through, run the Cleanup cell and then restart from Setup. The Cleanup cell:
- Kills the Flask server process
- Removes `app.py`

Then re-run all cells from the top.
