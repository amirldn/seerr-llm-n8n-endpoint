# Seerr LLM Request Agent

A natural language media request agent for [Seerr](https://github.com/fallenbagel/jellyseerr) / [Overseerr](https://overseerr.dev/). Ask it to download a movie or TV show in plain English and it handles the rest.

> *"Hey Siri, download Season 2 of Severance"*

Two implementations are included:

| | **n8n Workflow** | **Node.js Server** |
|---|---|---|
| Setup | Import a JSON file | Docker container or `npm start` |
| LLM | Google Gemini Flash | Gemini or Claude |
| Anime routing | Yes (auto-detect) | No |
| User/device logging | Yes | No |
| Dependencies | n8n instance | Node.js 20+ |
| File | `workflow.json` | `src/` |

---

## n8n Workflow (Recommended)

A zero-code n8n workflow that handles everything via webhook. Designed to be shared as an **Apple Shortcut** so friends can request media with a single tap — no Seerr account needed.

### How It Works

```
User: "download season 3 of Severance"
            |
    n8n Webhook (POST)
            |
    Gemini Flash (extract intent)
            |
    Seerr Search API
            |
    Check availability
            |
    Seerr Request API
            |
User: {"success": true, "message": "Requested 'Severance' season(s) 3"}
```

### Features

- Natural language parsing via Google Gemini Flash
- Automatic anime detection and routing to anime Sonarr/Radarr instances
- Skips already-available or already-requested media
- Logs requesting user and device name
- No authentication required on the webhook endpoint
- Returns clean JSON for Shortcut integration

### Setup

#### 1. Import the Workflow

1. Open your n8n instance
2. **Workflows** > **Import from File** > select `workflow.json`

#### 2. Create Credentials

**Google Gemini API Key:**
1. **Credentials** > **Add Credential** > search **"Google Gemini (PaLM)"**
2. Paste your [Gemini API key](https://aistudio.google.com/apikey)
3. Save

**Seerr API Key:**
1. **Credentials** > **Add Credential** > search **"Header Auth"**
2. **Name**: `X-Api-Key`
3. **Value**: your Seerr API key (Seerr > Settings > General)
4. Rename the credential to "Seerr API Key"
5. Save

#### 3. Wire Up Credentials

1. Open the imported workflow
2. Click **"Gemini Extract Intent"** node > select your Gemini credential
3. Click **"Seerr Search"** node > select your Seerr Header Auth credential
4. Click **"Seerr Request"** node > same Seerr credential
5. Save

#### 4. Update Seerr URL (if needed)

The workflow uses `http://seerr:5055` (Docker networking). Update in the **"Seerr Search"** and **"Seerr Request"** nodes if your setup differs.

#### 5. Activate

Toggle the workflow to **Active**. Webhook is live at:

```
https://your-n8n-domain.com/webhook/media-request
```

#### 6. Open the Webhook (if behind auth)

If n8n is behind a reverse proxy with authentication (e.g. Authelia), ensure `/webhook/` bypasses auth. With [Saltbox](https://docs.saltbox.dev), this is handled automatically.

### Usage

```bash
curl -X POST https://your-n8n-domain.com/webhook/media-request \
  -H "Content-Type: application/json" \
  -d '{"prompt": "download season 3 of Severance", "user": "Amir", "device": "Terminal"}'
```

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `prompt` | string | Yes | Natural language request |
| `user` | string | No | Requesting user (for logging) |
| `device` | string | No | Device name (for logging) |

#### Response

```json
{
  "success": true,
  "message": "Requested 'Severance' season(s) 3",
  "requestedBy": "Amir",
  "device": "iPhone"
}
```

#### Example Prompts

| Prompt | Result |
|--------|--------|
| `download Oppenheimer` | Requests the movie |
| `get me season 2 of The Bear` | Requests specific season |
| `add all of Breaking Bad` | Requests all seasons |
| `download demon slayer season 4` | Routes to anime Sonarr |
| `get the anime movie Suzume` | Routes to anime Radarr |

### Anime Routing

The LLM automatically detects anime titles and routes them to dedicated Sonarr/Radarr anime instances via Seerr's `serverId` parameter. To configure:

1. Add anime Sonarr/Radarr instances in Seerr (Settings > Services)
2. Note their server IDs
3. Update the IDs in the **"Process Results"** code node (`requestBody.serverId = 2`)

No separate anime instances? It still works — requests go to your defaults.

---

## Node.js Server (Original)

The original [OverseerAgent](https://github.com/omer182/OverseerAgent) by Omer S. — a standalone Node.js/TypeScript server.

### Setup

1. **Clone and install:**
   ```sh
   git clone https://github.com/amirldn/seerr-llm-n8n-endpoint.git
   cd seerr-llm-n8n-endpoint
   npm install
   ```

2. **Configure `.env`:**
   ```env
   LLM_PROVIDER=gemini
   LLM_API_KEY=your_gemini_api_key
   OVERSEERR_API_KEY=your_overseerr_api_key
   OVERSEERR_URL=http://your_seerr_instance
   ```

3. **Run:**
   ```sh
   npm run build && npm start
   ```

4. **API:** `http://localhost:4000/api/prompt`

### Docker Compose

```yaml
services:
  overseeragent:
    image: ghcr.io/omer182/overseeragent:latest
    container_name: overseeragent
    ports:
      - 4000:4000
    environment:
      - OVERSEERR_URL=http://seerr:5055
      - OVERSEERR_API_KEY=your_api_key
      - LLM_PROVIDER=gemini
      - LLM_API_KEY=your_llm_api_key
    restart: unless-stopped
```

---

## Apple Shortcut (Works with Both)

1. **Ask for Input** (Text): "What do you want to download?"
2. **Get Name of Device**
3. **Get Contents of URL**
   - Method: POST
   - URL: your webhook/API URL
   - Body (JSON):
     ```json
     {
       "prompt": "[input]",
       "user": "YourName",
       "device": "[device name]"
     }
     ```
4. **Get Dictionary Value**: key `message`
5. **Show Alert** / **Speak Text**: display the result

Share via iCloud link. Each user's device name is captured automatically.

---

## Credits

- Original [OverseerAgent](https://github.com/omer182/OverseerAgent) by [Omer S.](https://github.com/omer182) and [kobik](https://github.com/kobik)
- n8n workflow implementation by [Amir Maula](https://github.com/amirldn)

## License

MIT
