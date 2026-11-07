# Telegram Gemini n8n Bot

An AI assistant for Telegram built as a single n8n workflow. It answers text messages and voice notes using Google Gemini, remembers earlier messages in a chat, and behaves sensibly in groups by replying only when it is addressed.

- **Platform:** n8n, self-hosted on Selfhost.dev
- **AI model:** Google Gemini (Flash), called through HTTP Request nodes
- **Interface:** Telegram bot (`@srushti_n8n_gemini_bot`)
- **Workflow file:** [`telegram-gemini-bot.json`](telegram-gemini-bot.json)

## Features

| Feature | How it works |
|---|---|
| Text replies | Any text message in a private chat gets a Gemini answer. |
| Voice notes | The voice file is downloaded, converted to base64 and transcribed by Gemini, then answered like a text message. |
| Conversation memory | Earlier messages from the same chat are included in the next Gemini request, so the bot can answer questions such as "What is my name?". |
| Group behaviour | In groups the bot replies only when it is @mentioned or when someone replies to one of its messages. Other group messages are ignored. |
| Private chats | The bot always replies. |

## How the workflow works

```
Telegram Trigger
      |
 Is voice note? --yes--> Download voice file -> Voice to base64 -> Gemini transcribe --+
      |                                                                                |
      no                                                                               |
      +---------------------------------> Prepare message <---------------------------+
                                                |
                                         Should I reply?
                                          |           |
                                         yes          no -> (stop)
                                          |
                                    Build request
                                          |
                                     Gemini reply
                                          |
                              Save memory and format reply
                                          |
                                     Send reply
```

1. **Telegram Trigger** receives every `message` update from the bot's webhook.
2. **Is voice note?** sends voice messages through the transcription branch and text messages straight on.
3. **Prepare message** (Code node) produces one clean object: `chatId`, `text`, `inputType`, `isGroup` and `shouldReply`. The group rule lives here: `shouldReply = !isGroup || mentioned || repliedToBot`.
4. **Should I reply?** stops the run when `shouldReply` is false.
5. **Build request**, **Gemini reply** and **Save memory and format reply** build the prompt with history, call Gemini and store the exchange.
6. **Send reply** posts the answer back to the same chat.

## Setup

### Requirements
- An n8n instance reachable over HTTPS (Telegram webhooks need a public HTTPS URL)
- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- A Google Gemini API key from [Google AI Studio](https://aistudio.google.com/)

### Steps
1. **Create the bot.** In Telegram, message BotFather, send `/newbot` and save the token.
2. **Import the workflow.** In n8n choose *Import from file* and select `telegram-gemini-bot.json`.
3. **Add the Telegram credential.** Open the *Telegram Trigger* node, create a credential with your bot token, and select the same credential on *Download voice file* and *Send reply*.
4. **Add your Gemini key.** Put your API key in the *Gemini transcribe* and *Gemini reply* HTTP Request nodes. Never commit a real key to this repository.
5. **Set the bot username.** In the *Prepare message* node, change the line `const BOT = '...';` to your bot's username without the `@`.
6. **Check the model name.** In both Gemini HTTP nodes, make sure the URL ends with a model that exists for your key, for example `models/<model-name>:generateContent`.
7. **Allow group messages.** In BotFather send `/setprivacy`, choose your bot and select **Disable**.
8. **Publish the workflow.** Click *Publish*. This registers the Telegram webhook.

### Using it in a group
1. Add the bot to the group by typing its full username in the add-member search. Do not search by display name, because another bot may share it.
2. Send `@your_bot_username what is 2+2?`, or reply directly to one of the bot's messages.

## Testing

Tests run on 7 Nov 2026 against the published workflow.

| # | Test | Input | Result |
|---|---|---|---|
| 1 | Private text | "What is capital of india" | Correct answer, about 4 s |
| 2 | Private text | "what is 2+2" | Correct answer, about 4 s |
| 3 | Memory | "My name is Srushti", then "What is my name?" | Remembered the name |
| 4 | Voice note | Voice note asking for the capital of France | Transcribed and answered correctly |
| 5 | Group, @mention | `@srushti_n8n_gemini_bot hi` | Replied, about 3 s |
| 6 | Group, no mention | Plain message | Run stops at *Should I reply?* with no reply |

Execution times come from the n8n Executions list. Typical full replies take 1 to 6 seconds.

## What broke and how it was fixed

| Problem | Cause | Fix |
|---|---|---|
| First test run failed at the Gemini model node | Wrong or unavailable model name | Changed the model name in the Gemini nodes' URL |
| Group messages never reached n8n (11:41 AM to 1:16 PM) while private chat worked | Not fully diagnosed. Telegram's webhook info looked healthy (correct URL, 0 pending updates, no errors) | Group replies started working after the bot token was revoked and replaced, the workflow was republished, and the bot was removed from the group and added again |
| Turning off privacy mode, sending a `/start@bot` command and using a second new group did not help | Privacy mode was not the blocker | Not applicable |
| Publishing failed with "The resource you are requesting could not be found" | The n8n credential still held the revoked token | Pasted the new token into the credential, then published |
| Test button failed with "can't listen for test executions at the same time as production" | Telegram allows one webhook per bot, so test and production runs conflict | Tested with the published workflow instead |

The exact step that fixed group delivery is uncertain because several changes were made in a row. If group messages stop arriving after you change the token, remove the bot from the group and add it again.

## Security notes

- Never commit your bot token or Gemini key. Search the workflow JSON for `AIza`, `key=` and your token before pushing.
- If a token is ever exposed, send `/revoke` to BotFather, update the n8n credential, then unpublish and publish the workflow.
- n8n credentials are not included in an exported workflow, but keys typed directly into node fields are.

## Known limitations

- Telegram allows only one webhook per bot, so a bot token cannot be used by two active workflows at once.
- Memory is limited to recent messages and is lost if it is stored only in the workflow and that data is cleared.
- Group delivery depends on the bot being added correctly to the group.

## Author

Srushti Jain
