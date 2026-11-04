# Chaos checks (CareChaos simulation)

These files describe a **simulated** model of how Open Wearables emits outgoing webhooks, and run in CI with [CareChaos](https://github.com/batool-fariha/carechaos). They exercise a model, not the running application: no network calls, no real data, no Svix.

## What is modeled

Only the "Notify" step from the README: after data is saved, `emit_webhook_event` sends a message to Svix.

| Setting | Value | Source in this repo |
|---|---|---|
| Retries | 2 | `max_retries=2` in `backend/app/integrations/celery/tasks/emit_webhook_event_task.py` |
| Retry delay | 5 s, constant | `default_retry_delay=5` in the same task (no backoff configured) |
| Failure signal | `svix.send` returns `None` on a connection error, the task raises `self.retry(...)` | `backend/app/services/outgoing_webhooks/svix.py`, same task |
| After the last retry | the task fails; no dead-letter queue or redrive found in this task | `deadLetterRedrive: false` |

In CareChaos terms, the `delivery` stage stands for "the Svix call". Stage capacities are left at CareChaos defaults because this repo does not publish throughput numbers; calibrate them from your own Celery/Flower metrics before reading anything into latency or backlog results.

## The two checks

* `webhook-short-outage.json` (**blocking**): an 8 s Svix outage. With 2 retries at 5 s the sender should ride it out. If someone lowers `max_retries` or the delay so that this no longer holds, the check fails.
* `webhook-long-outage.json` (**informational**, `continue-on-error`): a 60 s outage. Under the settings above the model loses events raised in the first ~50 s, because retries run out at about 10 s and nothing re-queues them. This is a statement about the model and the current retry settings, not a verified production incident. It is here so the trade-off stays visible.

## Run locally

```bash
git clone https://github.com/batool-fariha/carechaos && cd carechaos && npm ci
npm run carechaos -- run /path/to/open-wearables/chaos/webhook-short-outage.json
```

If you change the retry settings in the task, update the numbers in these files to match. Format reference: [CareChaos experiment files](https://github.com/batool-fariha/carechaos/blob/main/docs/pipeline-file.md).
