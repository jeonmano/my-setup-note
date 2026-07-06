
# ubuntu update

```
minho@DESKTOP-3U0O7D9:~$ sudo apt-get update
```


# codex setup

```
curl -fsSL https://chatgpt.com/codex/install.sh | sh
```

# codex login

```
minho@DESKTOP-3U0O7D9:~$ codex login --device-auth

Welcome to Codex [v0.142.5]
OpenAI's command-line coding agent

Follow these steps to sign in with ChatGPT using device code authorization:

1. Open this link in your browser and sign in to your account
   https://auth.openai.com/codex/device

2. Enter this one-time code (expires in 15 minutes)
   R7H3-18IMO

Device codes are a common phishing target. Never share this code.

Successfully logged in
```

# hermes setup

```
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```


# slack manifest 생성

```
minho@DESKTOP-3U0O7D9:~$ hermes slack manifest --write
Slack manifest written to: /home/minho/.hermes/slack-manifest.json

Next steps:
  1. Open https://api.slack.com/apps and pick your Hermes app
     (or create a new one: Create New App → From an app manifest).
  2. Features → App Manifest → paste the contents of
     /home/minho/.hermes/slack-manifest.json
  3. Save; Slack will prompt to reinstall the app if scopes or
     slash commands changed.
  4. Make sure Socket Mode is enabled and you have a bot token
     (xoxb-...) and app token (xapp-...) configured via
     `hermes setup`.
```

# slakc 연계
- https://api.slack.com/apps 생성 후 bot id, app id, slack userid, slack channel id 사용해서 연계
- <img width="907" height="185" alt="image" src="https://github.com/user-attachments/assets/855b6b0c-c726-4a92-a17c-36b3a044388e" />


