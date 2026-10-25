# Sampling and model choice

Written 2026-10-25, dictated by Anurag; Claude tidied it up. Revise: 26 Oct, 1 Nov, 24 Nov, 23 Jan.
Lines marked **[gap]** are the parts to say more sharply next time.

## 1. How the next token is picked

- The model gives a score to every possible next token and turns the scores into probabilities.
- It draws **one** token, adds it to the text, then repeats, one token at a time.
- **[gap]** It is one token per step, not a "set of tokens" or several samples at once. Each step scores the whole vocabulary again.
- Java view: a `Map<Token, Double>` of weights plus a weighted random pick, in a loop.

## 2. Temperature

- Temperature changes how spread out the probabilities are before the draw.
- **Low (about 0):** the highest-scored token almost always wins, so output is stable and repeatable.
  Use it for JSON output, extraction and classification.
- **High (about 1):** weaker tokens get a real chance, so output varies more.
  Use it for creative writing and brainstorming.
- **[gap]** Temperature 0 is *nearly* deterministic, not guaranteed identical on every call.

## 3. Top-p (nucleus sampling)

1. Sort the tokens by probability, highest at the top.
2. Add up the probabilities from the top until the total reaches p (for example 0.9).
3. Drop everything below that point.
4. Draw the next token only from the tokens that are kept.

- Temperature changes the weights; top-p cuts off the unlikely tail.
- **[gap]** Tune one of the two, not both.

## 4. Temperature 0 does not stop hallucination

- Low temperature makes the model consistent, not correct. It can be consistently wrong.
- What reduces hallucination:
  - **Grounding:** put the real facts in the prompt (documents, database results) and tell the model to answer only from them.
  - Let it say **"I don't know"** when the facts aren't there.
  - **Check the output in code:** validate the JSON, check IDs and amounts against the database, ask for sources.
- **[gap]** "Give it better answers and feedback" was too vague. In an interview, name grounding, "I don't know" and checks in code.

## 5. Choosing a model

| Size | When | Example |
| --- | --- | --- |
| Small | Simple task, huge volume, speed and cost matter | Sort 2M transactions a day into 10 categories |
| Mid-size | Need some reasoning, balanced with speed and cost | "Explain this declined payment" for support agents |
| Large | Hard, complex reasoning; speed matters less | Complex or unclear cases sent up from the smaller models |

- **Routing:** send most traffic to the small model and escalate only the hard or unsure cases to bigger ones. This is like a cache in front of a slow database: most requests take the cheap path.
- **[gap]** How to decide what to escalate: low-confidence answers, failed validation, or "other" results.
- **[gap]** Latency also depends on **output length**: tokens come out one at a time, so cap `max_tokens` and ask for short answers. This is often a bigger speed win than changing the model.
