# Cost-aware model router

An [n8n](https://n8n.io) workflow that sends each question to the cheapest Claude model that can answer it.

**[View the live workflow →](https://ananmay1224.github.io/cost-aware-model-router/)**

![The workflow in the n8n editor](screenshot.png)

## Why

Most "ask your data" assistants send every question to the biggest model. In a cost-analytics workload most questions are simple lookups ("EC2 spend last month"), and those are the ones that overpay. This workflow puts a free, deterministic classifier in front of the models. Simple questions go to **Claude Haiku 5.5**; only the complex ones go to **Claude Opus 5.5**.

## How it works

```
POST /webhook/<id>  {"question": "..."}
        │
        ▼
 Classify complexity   (Python Code node: word count + hint words)
        │
        ▼
   Route by cost ── simple ──▶ Basic LLM Chain: Simple  ◀── Haiku 5.5
        │                               │
        └──── complex ─▶ Basic LLM Chain: Complex ◀── Opus 5.5
                                        │
                                        ▼
                                      Merge ──▶ Return answer
```

Both branches return the same shape (`[{ "text": "..." }]`), so the caller never needs to know which model answered.

## Routing rule

A question is **complex** if either of these is true:

- it has **more than 25 words**, or
- any word **starts with** one of these hints:
  `compare`, `trend`, `forecast`, `why`, `breakdown`, `across`, `anomaly`, `explain`

Otherwise it is **simple**. Matching is by prefix on whole words, so "trends" and "explained" match but a hint buried inside an unrelated word does not. A missing or blank `question` routes to simple and adds an `error` field instead of failing.

The classifier also outputs `word_count` and `matched_hints`, so you can explain every routing decision after the fact.

## Try it yourself

1. Run n8n locally with the [one-line setup](https://docs.n8n.io/deploy/host-n8n/install-options/one-line-setup).
2. In n8n, create a new workflow, open the **⋯** menu and choose **Import from File**, then select [`workflow.json`](workflow.json).
3. Open the **Haiku 5.5** and **Opus 5.5** nodes and attach your own Anthropic credential to each one.
4. **Publish** the workflow.
5. Copy the production URL from the **Question in** node and call it:

```bash
curl -X POST http://localhost:5678/webhook/<id> \
  -H "Content-Type: application/json" \
  -d '{"question": "Why is my EC2 spend so high?"}'
```

`/webhook-test/<id>` is for runs triggered from the editor (click **Execute** first). `/webhook/<id>` is for the published workflow.

## Limitations

- **This is a demo.** No billing data is connected. The models are told this and answer from general knowledge, naming the data they would need for a precise answer.
- **The classifier is a keyword heuristic.** It will misroute some questions. For example, "anomalies" does not start with "anomaly", and "comparison" does not start with "compare". That trade-off is deliberate: an LLM classifier in front of the router would cost more than routing saves on cheap questions.

## Roadmap

- Log every routing decision with its token cost per question.
- Measure answer quality on the cheap route, to catch questions that should have gone to Opus.
- Replace the heuristic with a trained classifier once there is labelled traffic.

## License

[MIT](LICENSE)
