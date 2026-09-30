# Nerve

A text-based choose-your-own-adventure that runs entirely on your own machine. One local
model narrates the story; [Nimble](https://ollama.com/library/nimble) — Bespoke Labs' 9B
decision model — rates every choice you make, and its own probability spread decides how
that choice lands.

No backend, no build step, no keys. One `index.html`.

## Running it

```sh
ollama pull nimble          # the judge (required)
ollama pull llama3.2:3b     # a narrator (any chat model works)

git clone https://github.com/Jonezzyboy/nerve && cd nerve
python3 -m http.server 8383
```

Then open <http://localhost:8383>. Ollama accepts browser calls from `localhost` on any
port out of the box, so nothing else needs configuring.

To play it from `https://jonezzyboy.github.io/nerve/` instead, Ollama has to be told that
origin is allowed. If you run it from a terminal, stop it and start it with:

```sh
OLLAMA_ORIGINS="https://jonezzyboy.github.io" ollama serve
```

If you run the **menu-bar app**, set the variable and then quit the app from the menu bar
and reopen it:

```sh
launchctl setenv OLLAMA_ORIGINS "https://jonezzyboy.github.io"
```

`killall ollama` is not enough on its own — the app immediately restarts the server with
its own environment, so the setting never reaches it. The symptom is a 403 on the CORS
preflight while `curl http://localhost:11434/api/tags` still works fine.

The **Models** button in the page picks the endpoint, the narrator, the judge and the run
length; everything is stored in `localStorage`. With no chat model installed, Nerve falls
back to a built-in story (*Nightshift*, at the Marlowe Hotel) — Nimble still judges every
choice, so the game is fully playable on the judge alone.

## How the judging works

Nimble cannot write prose. It answers typed questions about a block of state and returns a
probability distribution per question, through Ollama's `/v1/systemone` endpoint. Each turn
Nerve sends it the scene, your goal and your action, and asks five questions:

| Question | Type | What it decides |
|---|---|---|
| `nerve` | score | How much nerve the action shows, on a five-level rubric |
| `judgement` | score | How well the action reads the scene |
| `approach` | choice | force / guile / words / caution / strange |
| `danger` | noul | Whether it exposes you to immediate harm |
| `progress` | noul | Whether it moves you towards the goal |

The two score questions return a full spread, not a single answer — and Nerve **samples**
from that spread rather than taking the top level. A confident read plays out the way it
read; a torn one is a genuine coin toss. That sampled pair, pushed around by `danger` and
`progress`, picks one of five outcome bands (disaster → triumph), and the band is handed to
the narrator as a hard constraint on what it is allowed to write next. The panel shows you
the raw distributions, so you can see exactly where the story turned.

Free-typed actions are judged the same way as the offered ones — Nimble answers in a second
or two, so there is no reason to restrict you to three buttons.

`approach` is tallied across the run, and at the end Nimble grades the whole thing the same
way it graded each turn: a title (choice) and how well you came out of it (score).

## Notes

- Nimble's context window is 8k and the request body caps at 64 KiB, so only the scene, the
  goal and the current action are sent — not the whole history.
- Any chat model can narrate, but small ones drift out of JSON. If scenes start failing,
  switch narrator in **Models**.
- Nimble scores rubrics less accurately than it picks from a list (54.6% vs 81.6% on
  Bespoke's benchmarks). That is fine here — the sampling makes the uncertainty part of the
  game rather than a defect.
