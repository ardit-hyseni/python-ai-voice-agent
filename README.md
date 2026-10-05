# Pharmacy voice agent

A phone-callable pharmacy assistant. Twilio receives the call, this Python server bridges the audio, and Deepgram's Voice Agent API handles speech-to-text, the reply, and text-to-speech. Mid-call the agent can look up drugs, place orders, and check order status through local Python functions.

Orders and the drug catalog live in memory inside `pharmacy_functions.py`. Restarting the server clears them.

See [docs/architecture.md](docs/architecture.md) for the call flow.

## Requirements

- Python 3.13 or newer
- [uv](https://docs.astral.sh/uv/)
- A [Deepgram](https://deepgram.com) account and API key. Deepgram runs the speech models and `gpt-4o-mini`, so a separate OpenAI key is not required.
- A [Twilio](https://www.twilio.com) account with a **local US voice number** (an area code such as 415 or 212). Toll-free numbers (800, 888, 877, 866, 855, 844, 833) cannot be called from outside the US and Canada.
- The [ngrok](https://ngrok.com) CLI, authenticated with `ngrok config add-authtoken <token>`
- A phone that can dial that US number. From outside the US, dial `+1` and the 10-digit number. On a Twilio trial, add your phone under **Phone numbers → Verified caller IDs** first.

## Setup

```bash
uv sync
```

Create a `.env` file in the project root:

```
DEEPGRAM_API_KEY=your_key_here
```

`config.json` holds the agent prompt, greeting, voice, and tool schemas. Edit that file to change how the assistant speaks.

## Run

Start the tunnel and the server in two terminals. ngrok must be the CLI, not the `ngrok` package from pip.

```bash
ngrok http 5000
```

```bash
uv run python main.py
```

The server listens on `ws://localhost:5000` and prints `Started server.`

## Connect Twilio

1. Copy the `https` forwarding URL from ngrok.
2. In the Twilio console, create a TwiML Bin and paste:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
    <Say>This call may be monitored or recorded.</Say>
    <Connect>
        <Stream url="wss://YOUR-NGROK-HOST/twilio" />
    </Connect>
</Response>
```

Use `wss://`, not `https://`, and keep a single `<?xml ...?>` line. Replace the host with your ngrok host.

3. Open the phone number → Voice. Set **A call comes in** and **Primary handler fails** to that TwiML Bin. Save.

A free ngrok URL changes every time ngrok restarts. Update the bin when it does, or Twilio will stream the call to a dead address.

Call the number. A successful call prints `get our streamsid` in the server log, then the pharmacy greeting.

## What the agent can do

| Tool | Purpose |
| --- | --- |
| `get_drug_info` | Price, description, and pack size for a drug in the catalog |
| `place_order` | Store an order under the caller's name. Quantity and price come from the catalog. |
| `lookup_order` | Read an order back by its numeric id |

## Project layout

| File | Role |
| --- | --- |
| `main.py` | WebSocket server and Twilio ↔ Deepgram audio bridge |
| `config.json` | Audio format, prompt, greeting, and tool schemas |
| `pharmacy_functions.py` | Drug catalog, orders, and the functions the agent calls |
| `.env` | `DEEPGRAM_API_KEY` (not committed) |
