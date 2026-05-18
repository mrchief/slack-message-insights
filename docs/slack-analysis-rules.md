# Slack Message Analysis Rules

## Source Files

- Channel inventory: `info/channels-T01E0D9F1NW.txt`
  - Maps Slack channel IDs to channel, group, or DM names.
  - Includes an archive marker in the `Arch` column.
- Target channel list: `info/channel list.txt`
  - Whitespace-separated Slack channel IDs selected for analysis.
  - A target channel may be listed here before a matching dump exists.
- Slack message dumps: `dumps/*.json`
  - Each dump is expected to be a Slack-style JSON object with `channel_id`, `name`, and `messages`.
  - Dump filenames are expected to match the channel ID, for example `dumps/C01E0D9FL2E.json`.

## Analysis Scope

- Primary user ID: `U05BV05GYQH`.
- Default time window: the past 90 days from the configured end date.
- Default target channels: all channel IDs in `info/channel list.txt`.
- Channels listed in `info/channel list.txt` but missing from `dumps/` are reported as warnings.
- Empty dump files are valid and are included in summary counts.

## Message Selection Rules

- A message is relevant when it was authored by `U05BV05GYQH`.
- Both top-level messages and threaded replies authored by `U05BV05GYQH` are included.
- Slack system messages such as joins/leaves are ignored unless they are already part of a selected context window.
- Reactions by `U05BV05GYQH` are not treated as authored messages.

## Conversation Grouping Rules

- Threaded conversations are grouped by `channel_id + thread_ts`.
- If `U05BV05GYQH` replies inside a thread, the entire available thread is included:
  - the root message,
  - all inline replies with the same `thread_ts`,
  - and any embedded `slackdump_thread_replies` present in the dump.
- Top-level non-thread messages by `U05BV05GYQH` are grouped with surrounding top-level messages from the same channel.
- Nearby top-level context windows are merged when they overlap or are close together, so agents see a coherent local conversation instead of repeated fragments.
- Each conversation group receives a short `topic_hint` derived from the root/first meaningful message and attachment titles. This is a lightweight label, not an AI summary.

## Context Rules

- Default top-level context: 4 messages before and 4 messages after each relevant top-level message.
- Default cluster merge gap: 30 minutes between top-level messages.
- Thread context is preferred over top-level context when a message belongs to a thread.
- Message text, attachment titles, attachment text, file titles, reaction names, timestamps, users, channel names, and thread metadata are preserved where available.

## Outputs

- Excel workbook: `output/slack_message_analysis.xlsx`
  - `Summary`: run configuration, counts, and warnings.
  - `Conversations`: one row per grouped conversation.
  - `Messages`: one row per extracted message.
  - `My Messages`: one row per authored message by `U05BV05GYQH`.
- Agent context sidecar: `output/conversation_contexts.jsonl`
  - One JSON object per conversation group with ordered messages and transcript text.

## Reproducibility

Run the analyzer from the repository root:

```bash
python3 -m pip install -r requirements.txt
python3 src/analyze_slack_messages.py --end-date 2026-05-18
```

Useful options:

```bash
python3 src/analyze_slack_messages.py \
  --user-id U05BV05GYQH \
  --months-days 90 \
  --context-before 4 \
  --context-after 4 \
  --merge-gap-minutes 30
```
