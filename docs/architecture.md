# Architecture

Pharmacy voice agent: Twilio telephony, a local Python WebSocket bridge, and Deepgram's Voice Agent API for speech-to-speech.

## System architecture

```mermaid
flowchart TB
    Caller["Caller phone"]

    subgraph Cloud["Cloud services"]
        Twilio["Twilio<br/>phone number + Media Streams"]
        subgraph DG["Deepgram Voice Agent API<br/>wss://agent.deepgram.com/v1/agent/converse"]
            STT["Listen / STT<br/>Deepgram nova-3"]
            LLM["Think / LLM<br/>OpenAI gpt-4o-mini"]
            TTS["Speak / TTS<br/>Deepgram aura-2-thalia-en"]
            STT --> LLM --> TTS
        end
    end

    subgraph Dev["Local development"]
        Ngrok["ngrok<br/>HTTPS / WSS tunnel"]
        UV["uv + PyCharm<br/>Python 3.13 env"]

        subgraph App["voice-agent  ·  localhost:5000"]
            Main["main.py<br/>asyncio + websockets"]

            subgraph Bridge["Per-call audio bridge  ·  twilio_handler"]
                TR["twilio_receiver<br/>start / media / stop<br/>buffer 20×160 mulaw"]
                SS["sts_sender<br/>forward chunks to Deepgram"]
                SR["sts_receiver<br/>TTS audio + JSON events"]
                BI["handle_barge_in<br/>UserStartedSpeaking → Twilio clear"]
                FC["handle_function_call_request<br/>FunctionCallRequest → local Python"]
            end

            Config["config.json<br/>mulaw 8 kHz · prompt · tool schemas · greeting"]
            Env[".env<br/>DEEPGRAM_API_KEY"]
            Pharmacy["pharmacy_functions.py"]

            subgraph Store["In-memory stores"]
                DrugDB["DRUG_DB"]
                OrdersDB["ORDERS_DB"]
            end
        end
    end

    Caller -->|"PSTN call"| Twilio
    Twilio -->|"Media Stream WebSocket"| Ngrok
    Ngrok -->|"WSS → ws://localhost:5000"| Main
    Main --> TR
    TR -->|"inbound mulaw chunks"| SS
    SS -->|"binary audio"| STT
    TTS -->|"binary mulaw TTS"| SR
    SR -->|"event: media  base64 mulaw"| Twilio
    Twilio -->|"playback"| Caller

    LLM -->|"FunctionCallRequest JSON"| SR
    SR --> FC
    FC --> Pharmacy
    Pharmacy --> DrugDB
    Pharmacy --> OrdersDB
    FC -->|"FunctionCallResponse"| LLM

    LLM -->|"UserStartedSpeaking"| SR
    SR --> BI
    BI -->|"event: clear"| Twilio

    Config -->|"Settings on connect"| DG
    Env -->|"auth subprotocol token"| DG
    UV -.-> App
```

## Call workflow

```mermaid
sequenceDiagram
    autonumber
    participant Phone as Caller
    participant Twilio as Twilio
    participant Ngrok as ngrok
    participant Server as main.py :5000
    participant DG as Deepgram Agent
    participant Tools as pharmacy_functions.py

    Phone->>Twilio: Dial provisioned number
    Twilio->>Ngrok: Open Media Stream WebSocket
    Ngrok->>Server: twilio_handler(twilio_ws)

    Server->>DG: Connect wss://agent.deepgram.com
    Server->>DG: Send config.json Settings<br/>(audio, prompt, functions, greeting)

    par Concurrent asyncio tasks
        Twilio->>Server: event start  streamSid
        Server->>Server: streamsid_queue
    and
        loop Inbound speech
            Phone->>Twilio: Voice (mulaw)
            Twilio->>Server: event media  inbound payload
            Server->>Server: Decode base64, buffer to 20×160
            Server->>DG: Binary mulaw chunk
        end
    and
        DG->>Server: Greeting / TTS mulaw
        Server->>Twilio: event media  base64 payload
        Twilio->>Phone: Play audio
    end

    opt Interruption
        DG->>Server: UserStartedSpeaking
        Server->>Twilio: event clear  streamSid
        Note over Twilio,Phone: Stop current playback immediately
    end

    opt Tool / function calling
        DG->>Server: FunctionCallRequest
        Server->>Tools: get_drug_info / place_order / lookup_order
        Tools-->>Server: JSON result
        Server->>DG: FunctionCallResponse
        DG->>Server: Spoken reply as mulaw TTS
        Server->>Twilio: event media
        Twilio->>Phone: Play answer
    end

    Phone->>Twilio: Hang up
    Twilio->>Server: event stop
    Server->>Server: Close Deepgram + Twilio sockets
```

## File-to-role map

```mermaid
flowchart LR
    subgraph Repo["Repository"]
        M["main.py"]
        P["pharmacy_functions.py"]
        C["config.json"]
        E[".env"]
        PY["pyproject.toml / uv.lock"]
    end

    subgraph Roles["Runtime roles"]
        R1["WebSocket server + audio bridge"]
        R2["Pharmacy tools + in-memory DB"]
        R3["Agent personality, STT/LLM/TTS, tool schemas"]
        R4["Deepgram API auth"]
        R5["Deps: websockets, python-dotenv"]
    end

    M --> R1
    P --> R2
    C --> R3
    E --> R4
    PY --> R5
```

## Tech stack

| Layer | Role |
| --- | --- |
| Python (`asyncio` & `websockets`) | Asynchronous backend managing real-time audio socket streams |
| Twilio | Telephony: inbound calls and bidirectional audio via WebSockets |
| Deepgram Voice Agent API | End-to-end speech: STT, LLM reasoning, TTS |
| ngrok | Tunnels local Python server to a public HTTPS/WSS URL for Twilio |
| PyCharm & uv | Python environment setup and dependency management |

## Workflow

1. **Call handling** — A phone call hits a Twilio-provisioned number, which routes the audio stream to the Python server via WebSockets.
2. **Audio bridging** — The Python backend receives inbound mulaw buffers from Twilio and forwards them to Deepgram's WebSocket API in real time.
3. **Speech-to-speech** — Deepgram transcribes speech, feeds text into an LLM with custom prompt instructions, and generates synthetic voice audio.
4. **Real-time playback** — Synthesized audio chunks go back through the Python bridge to Twilio, which streams them to the caller's phone with minimal delay.

## Key functionality

- **Tool / function calling** — The voice agent can execute local Python functions mid-conversation (`get_drug_info`, `place_order`, `lookup_order`) against in-memory drug and order stores.
- **Interruption & stream handling** — Real-time audio chunking and media stream events (`UserStartedSpeaking` → Twilio `clear`) keep latency low and playback stable when the caller talks over the agent.
