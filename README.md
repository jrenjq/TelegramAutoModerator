# MiniChatSentryBot

> A stateless, deploy-anywhere Telegram admin bot written in Go.
> Uses the Claude API to automatically delete scam and inappropriate messages.

![Go](https://img.shields.io/badge/Go-1.x-00ADD8?logo=go&logoColor=white)
![Telegram Bot API](https://img.shields.io/badge/Telegram-Bot%20API-26A5E4?logo=telegram&logoColor=white)
![Claude 3.5 Sonnet](https://img.shields.io/badge/Claude-3.5%20Sonnet-D97757?logo=anthropic&logoColor=white)
![Version](https://img.shields.io/badge/version-v0.2-blue)
![Database](https://img.shields.io/badge/database-none-lightgrey)

---

## Table of Contents
1. [About the Project](#1-about-the-project)
2. [What Can It Do?](#2-what-can-it-do)
3. [How Does the Code Work?](#3-how-does-the-code-work)
4. [Implementation in Go](#4-implementation-in-go)
    - 4.1. [Design goals](#41-design-goals)
    - 4.2. [Project structure](#42-project-structure)
    - 4.3. [How an `[automod]` run flows](#43-how-an-automod-run-flows)
    - 4.4. [Go techniques used](#44-go-techniques-used)
    - 4.5. [Configuration](#45-configuration)
    - 4.6. [What I learned](#46-what-i-learned)
5. [Can I Just Invite This Bot into My Telegram Group?](#5-can-i-just-invite-this-bot-into-my-telegram-group)
6. [Other Documents](#6-other-documents)

---

## 1. About the Project

**[GitHub Repository](https://github.com/jrenjq/MiniChatSentryBot)**

- A Telegram bot, programmable to perform admin tasks.
- Written in Go. It aims to be **stateless** (no database needed) and **deploy-anywhere** (all it needs is internet access).
- Originally developed to manage a university Telegram group.

---

## 2. What Can It Do?

| Version | Feature | Tag | Status |
|---|---|---|---|
| `v0.2` | Deletes scam and inappropriate messages using the Claude 3.5 Sonnet API | `[automod]` | ✅ Active |
| `v0.1` | Finds messages with **3 or more** "👎" reactions and **deletes** them | `[downvote]` | ❌ Defunct |

> [!WARNING]
> `[downvote]` is **currently defunct**: Telegram made undocumented Bot API changes that removed reactions from update responses.

---

## 3. How Does the Code Work?

First and foremost, take a gander at the [Telegram Bot documentation](https://core.telegram.org/bots).

**On Telegram:**
1. Create a bot with `@BotFather`.
2. Invite the bot into your group as an admin who can access messages.
3. The bot gets its own URL on the Telegram server (see [`API_REFERENCE`](API_REFERENCE.md)).

**In your code:**
1. The bot calls its URL with a path for the information it wants (e.g. `/getUpdates`) and gets that information back as an HTTP response.
2. The bot does something in response (e.g. deletes a message).
3. The bot waits a while before calling its URL again, then repeats from step 1.

```mermaid
sequenceDiagram
    participant Bot as MiniChatSentryBot
    participant TG as Telegram Bot API
    Bot->>TG: GET /getUpdates
    TG-->>Bot: JSON with recent messages
    Bot->>Bot: decide what to act on
    Bot->>TG: GET /deleteMessage (chat_id, message_id)
    TG-->>Bot: ok: true
    Note over Bot: wait, then call again
```

---

## 4. Implementation in Go

### 4.1. Design goals

| Goal | What it means |
|---|---|
| **Stateless and deploy-anywhere** | One compiled binary, plain-text config, no database or web server |
| **One pass per run** | The program fetches updates, acts on them, and exits. Run it on a schedule (cron, systemd timer, etc.) |
| **Features toggled by config, not code** | Turn `[downvote]` and `[automod]` on or off in `.conf` without rebuilding |

### 4.2. Project structure

```
code/
├── main.go     # entrypoint: wires the features together
├── top/        # feature-level orchestration (one file per feature)
├── repo/       # "get X / do Y" functions that talk to Telegram, Claude, or files
├── structs/    # types that JSON responses are decoded into
└── utils/      # generic helpers: HTTP, env loading, file append, counting
```

Dependencies flow one way: `main → top → repo → utils/structs`. Each feature in `top/` reads like a recipe, and the details live in `repo/` and `utils/`.

### 4.3. How an `[automod]` run flows

```mermaid
flowchart TD
    A["main.go (single run)"] --> B["GET /getUpdates from Telegram"]
    B --> C["Extract messages with text or caption"]
    C --> D["Drop messages already in processedMessages.log"]
    D --> E["Claude rates each message: scam + inappropriateness (1-10)"]
    E --> F{"rating >= threshold?"}
    F -- yes --> G["GET /deleteMessage"]
    F -- no --> H["Leave message alone"]
    G --> I["Append all processed message uids to log"]
    H --> I
```

### 4.4. Go techniques used

| # | Technique | Where |
|---|---|---|
| 1 | Typed JSON decoding into structs | `structs/`, `utils/http.go` |
| 2 | Config from `.env` and `.conf` | `utils/env.go` |
| 3 | Structured LLM output parsed with a regex | `top/gptReviewFeature.go` |
| 4 | Claude API via plain `net/http` | `utils/claude.go` |
| 5 | Statelessness with an append-only log | `utils/file.go` |

#### 1. Typed JSON decoding
Go needs a struct for every JSON shape, so `structs/` mirrors Telegram's and Claude's responses. A single helper, `utils.Http_GET_JSON`, takes a URI and a pointer to any struct (`interface{}`) and decodes the response straight into it with `json.NewDecoder`, so every Telegram call reuses it.

#### 2. Config from the environment
Secrets and endpoints live in `.env`, and feature switches and thresholds live in `.conf`. Both are loaded with `godotenv`, and a missing key fails fast with a clear error.

#### 3. Structured output from an LLM
The system prompt makes Claude reply in a fixed `(A)(B):(X)(Y)` format: scam rating, inappropriateness rating, then a short reason for each. A regex pulls the parts out into a struct:

```go
re := regexp.MustCompile(`\((.*?)\)`)
matches := re.FindAllStringSubmatch(claude_rating_string, -1)
scam_rating_int, _ := strconv.Atoi(matches[0][1])
```

A message goes on the "naughty list" if either rating meets its configured threshold (`SCAM_RATING_GTE`, `INAPPROPRIATE_RATING_GTE`).

#### 4. Calling the Claude API with plain `net/http`
There's no SDK. The request is a `ClaudePayload` struct encoded to JSON, sent as a POST with `x-api-key` and `anthropic-version` headers, using an `http.Client` with a timeout.

#### 5. Statelessness without a database
Telegram keeps showing the bot recent messages, including ones it already deleted. To avoid re-rating them, every processed message gets a unique id (`chat_id:message_id`), appended to `processedMessages.log`. On the next run the log is loaded into a set and used to skip those messages:

```go
alr_processed_msgs_set := make(map[string]struct{})
for _, del_msg := range *previously_deleted_messages {
    alr_processed_msgs_set[del_msg] = struct{}{}
}
```

A `map[string]struct{}` gives O(1) lookups, and the empty struct costs no memory.

### 4.5. Configuration

| File | Key | Purpose |
|---|---|---|
| `.conf` | `GPT_REVIEW_FEATURE_ON` | Turn `[automod]` on or off |
| `.conf` | `SCAM_RATING_GTE` / `INAPPROPRIATE_RATING_GTE` | Deletion thresholds (1-10) |
| `.conf` | `*_DEBUG_MODE` | Verbose logging per feature |
| `.env` | `BOT_TOKEN`, `CLAUDE_API_KEY`, endpoint paths | Secrets and URLs (git-ignored) |
| `claude/prompts/systemPrompt.txt` | System prompt | Kept private so spammers can't tailor messages to it |

See [`SETUP`](SETUP.md) for the full list of values and how to create these files.

### 4.6. What I learned

See [`DEVLOG` → Food For Thought](DEVLOG.md#3-food-for-thought) for notes on:
- why Go needs structs for JSON decoding,
- structs vs. maps, and
- flattening unique struct fields into strings to simplify comparisons.

---

## 5. Can I Just Invite This Bot into My Telegram Group?

> [!CAUTION]
> **NO.** Create your own bot and host your own copy of the code.

You should **CREATE YOUR OWN BOT** via Telegram's `@BotFather`, get your own **BOT API TOKEN**, and host the code online to manage this bot (if you really want to).

You *could* invite this bot as an existing instance on Telegram into your own Telegram group. And you will have to make it an admin, where it'll be able to do many, many things indeed.

But you will **HAVE NO CONTROL OVER THE BOT**, because you don't have the **BOT API TOKEN**. Sometimes in the corner of your eye, you see the bot ... twitching ... talking to itself ... mumbling darkly as it does its job.

The bot that is an admin in your group that you can't control. You can't know for sure what it'll do. And people who make Telegram bots rarely pass the opportunity to play tricks on silly folks.

Don't be a silly folk :) Fork the code and host your own bot.

---

## 6. Other Documents

| Document | Contents |
|---|---|
| [`API_REFERENCE`](API_REFERENCE.md) | APIs used and explanations |
| [`DEVLOG`](DEVLOG.md) | Work done and tech debt |
| [`SETUP`](SETUP.md) | How to set up this project, for development and deployment |
