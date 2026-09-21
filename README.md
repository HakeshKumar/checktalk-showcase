# CheckTalk

### Live AI chess commentary backed by deterministic analysis and an event-driven AWS data platform

[Play the live demo](https://checktalk.hakeshk.com) · [Read the in-app project story](https://checktalk.hakeshk.com/#project)

![CheckTalk gameplay with Stockfish, commentary controls, and voice profiles](screenshots/gameplay.png)

## The project

CheckTalk turns a browser chess game into a live broadcast. A player faces Stockfish while the system identifies the opening, evaluates each move, tracks how both sides are playing, and delivers concise spoken commentary through three original voice profiles.

The visible chess experience sits on top of a production-oriented system designed around two different latency requirements:

- The **player path** returns grounded text and audio quickly enough to support a live game.
- The **analytics path** captures ordered lifecycle events without delaying gameplay.

I designed and built the product end to end: interaction design, chess engine integration, commentary orchestration, generative AI grounding, voice delivery, event streaming, analytics storage, infrastructure, deployment, and observability.

## Product experience

- Play Stockfish at multiple difficulty levels and time controls.
- Hear opening-aware, position-aware commentary for both players.
- Choose Host, Strategist, or Storyteller delivery styles.
- Keep commentary natural with queueing, stale-event coalescing, repetition controls, and deliberate pauses.
- Switch board and application themes.
- Review and replay the last three games from the current session.
- Continue playing if AI commentary, speech, or telemetry becomes unavailable.

![CheckTalk project details page](screenshots/project-details.png)

## Architecture

![CheckTalk production architecture](docs/architecture.svg)

### Player and commentary path

```text
Route 53 → CloudFront → private S3-hosted React application
                              │
                              ├─ Stockfish WASM evaluates locally
                              │
                              └─ API Gateway → Lambda → Bedrock + Polly
```

Stockfish establishes the chess facts. A deterministic backend validates the request, recomputes the evaluation delta, classifies the move, and derives position context. Amazon Bedrock writes only the short broadcast line; Amazon Polly synthesizes its selected delivery profile.

This separation keeps tactical judgement out of the language model and reduces confident but incorrect chess commentary.

### Streaming and analytics path

```text
Browser events → API Gateway → ingestion Lambda → Kinesis
                                                   │ game_id partition key
                                                   ↓
                                         persistence Lambda
                                                   ↓
                                  encrypted, versioned S3 Bronze
                                                   ↓
                                        Glue Catalog + Athena
```

Analytics is intentionally asynchronous. Events within a game remain ordered, separate games scale independently, failed records use bounded retries and a dead-letter queue, and the raw event history remains replayable and queryable.

## Engineering decisions

| Decision | Reason |
| --- | --- |
| Stockfish decides; the model narrates | Preserves chess correctness while retaining expressive language. |
| Finish active speech before switching topics | Avoids the unnatural mid-sentence cuts common in event-per-move audio. |
| Coalesce stale normal moves | Keeps commentary near the current board position when play accelerates. |
| Permit intentional pauses | Continuous filler became repetitive; a broadcast should know when silence is better. |
| Keep analytics off the player path | Telemetry failure cannot block a move or commentary response. |
| Use HTTP before WebSockets | Request/response is sufficient for a single-player experience; server push can wait for spectators or shared games. |
| Use Athena instead of always-on analytics compute | Matches a portfolio-scale, bursty query workload with a low idle cost. |

## Reliability and production practices

- Private S3 origin behind CloudFront, Route 53, ACM TLS, and security headers.
- Strict API validation, bounded payloads, throttling, and explicit CORS.
- Text fallback when speech synthesis or browser autoplay fails.
- Structured logs, CloudWatch metrics, dashboards, and alarms.
- Encrypted and versioned data storage with lifecycle policies.
- Partial batch failure handling, retry bisection, and an SQS dead-letter queue.
- GitHub OIDC deployment with short-lived AWS credentials rather than stored access keys.
- Terraform-managed infrastructure with explicit production deployment review.

## Verification

- **39 backend unit tests** across commentary, opening recognition, position context, ingestion, and persistence.
- Frontend static analysis and TypeScript production builds.
- Terraform configuration validation.
- Production smoke tests for each commentary profile and audio response.
- Graceful-degradation checks for commentary, speech, and telemetry failures.

## Technology

**Frontend:** React, TypeScript, chess.js, Stockfish WASM, Vite  
**AI and voice:** Amazon Bedrock, Amazon Nova Lite, Amazon Polly  
**Backend:** Python, AWS Lambda, API Gateway  
**Data:** Amazon Kinesis Data Streams, S3, AWS Glue, Athena  
**Platform:** CloudFront, Route 53, ACM, CloudWatch, SQS, Terraform, GitHub Actions/OIDC

## What I would build next

- Spectator mode with shared live games and server-pushed commentary.
- Evaluation and latency dashboards built from production telemetry.
- Human-versus-human rooms and authenticated cross-device game history.
- Automated commentary quality evaluation for chess accuracy, repetition, and timing.

## Repository scope

This public repository is an intentionally focused project showcase. It contains product screenshots and system-design documentation, but not the deployable application source, production infrastructure, environment configuration, or operational runbooks. The complete implementation is maintained privately and can be walked through during an interview.

---

Built by [Hakesh Kumar](https://github.com/HakeshKumar). The live application is available at [checktalk.hakeshk.com](https://checktalk.hakeshk.com).
