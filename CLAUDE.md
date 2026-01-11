# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

**Live app:** https://whatsapp-chat-analyzer-production-9d30.up.railway.app/

**Local development:**
```bash
uv sync
uv run python chat_analyzer_web.py
```
The app automatically finds a free port starting at 8080 (avoids macOS AirPlay conflict on 5000).

**Required environment variables:**
- `OPENROUTER_API_KEY` - OpenRouter API key (required for AI analysis)

**Optional environment variables:**
- `OPENROUTER_MODEL` - Model to use (defaults to 'anthropic/claude-3.5-haiku')
- `SECRET_KEY` - Flask session secret (defaults to dev key in development)
- `PORT` - Server port (if set, uses exact port; otherwise finds free port starting at 8080)
- `GOOGLE_TOKEN` - Google OAuth token JSON as string (for Google Docs/Drive URL analysis)

**Optional features:**
- URL content analysis requires Playwright browser: `uv run playwright install chromium`
- Google Docs/Drive analysis requires GOOGLE_TOKEN environment variable (see GOOGLE_SETUP.md)

## Architecture Overview

This is a Flask web app that analyzes WhatsApp chat exports using Claude Haiku 3.5 via OpenRouter. The analysis pipeline follows a two-stage approach:

### Analysis Pipeline

```
WhatsApp .txt file
    ↓
ChatParser (chat_analyzer.py) → Parses message format
    ↓
Date filtering → Filters to last N days from most recent message
    ↓
CandidateExtractor (chat_analyzer.py) → Pattern matching to find candidates
    ↓
AIAnalyzer (chat_analyzer_ai.py) → Claude API refines results
    ↓
AIMarkdownFormatter (ai_formatter.py) → Outputs markdown/HTML
    ↓
Download file
```

### Key Components

**chat_analyzer_web.py** - Flask entry point
- Handles file uploads at `/analyze` endpoint
- Routes queries to appropriate extractors
- Manages temporary file storage
- Serves markdown files for most query types, HTML for check-ins

**chat_analyzer.py** - Pattern matching layer
- `ChatParser`: Parses WhatsApp export format `[DD/MM/YYYY, HH:MM:SS] Sender: Message`
- `CandidateExtractor`: Pattern matches messages to find potential items
- Supports 8 query types: actions, urls, decisions, meetings, questions, deadlines, assignments, checkins

**chat_analyzer_ai.py** - AI refinement layer
- `AIAnalyzer`: Uses Claude Haiku 3.5 (via OpenRouter) to refine pattern-matched candidates
- Supports any OpenRouter-compatible model via `OPENROUTER_MODEL` environment variable
- Processes in chunks (max_tokens=8192) to avoid truncation
- Returns structured JSON with extracted information

**ai_formatter.py** - Output formatting
- `AIMarkdownFormatter`: Formats results with emojis, priorities, status indicators
- `format_actions_html()`, `format_urls_html()`, `format_checkins_html()`: Generate interactive HTML with SVG graphs
- Groups results by date, person, priority

**url_summarizer.py** - URL content extraction (optional feature)
- `analyze_url()`: Fetches and summarizes content from URLs
- Supports Google Docs/Drive (via Google API), ChatGPT shares, LinkedIn profiles (via Playwright)
- Falls back to BeautifulSoup for regular web pages

**google_analyzer.py** - Google API integration (optional feature)
- `get_credentials()`: Handles OAuth for Google APIs, supports both file-based (local) and env var (Railway)
- Railway deployment: Reads credentials from `GOOGLE_TOKEN` environment variable (JSON string)
- Local development: Uses token.json file (created via OAuth flow with credentials.json)
- google_analyzer.py:36 strips newlines from env var to handle Railway's multi-line text input
- Provides Drive and Docs service clients for URL analysis

## Important Implementation Details

### Date Filtering Logic
Date filtering is relative to the **most recent message in the file**, not today's date. This ensures old exports still return useful data:

```python
most_recent_date = max(datetime.strptime(m['date'], '%d/%m/%Y') for m in messages)
cutoff_date = most_recent_date - timedelta(days=days)
messages = [m for m in messages if datetime.strptime(m['date'], '%d/%m/%Y') >= cutoff_date]
```

### AI Token Limits
- Claude Haiku 3.5 (via OpenRouter): max_tokens=8192
- Candidates are chunked (50 per chunk by default) to avoid JSON truncation
- Previously had bug where max_tokens=4096 caused truncated responses

### Check-ins Feature
The check-ins query type is special:
- Returns HTML (not markdown) with interactive SVG graph
- Pattern matches multiple mood score formats: `X/10`, `- X`, `mood: X`
- AI extracts: person, date, time, score (0-10), comments
- Graph shows trends over time with hover tooltips

### Output Format
All query types now return HTML (not markdown as originally designed):
- chat_analyzer_web.py:168 always sets output_extension='html'
- Actions: Interactive HTML with expandable cards and filters
- URLs: HTML with link previews and content summaries
- Check-ins: SVG graphs embedded in HTML
- HTML files served inline in browser (chat_analyzer_web.py:273-280)

### WhatsApp Export Format
Expects exact format: `[DD/MM/YYYY, HH:MM:SS] Sender: Message`
- Multi-line messages are concatenated
- System messages have sender='SYSTEM'

### URL Content Analysis (Optional)
When analyzing URLs, the app can fetch actual content from links:
- For Google Docs/Drive: Uses Google API to extract text content
- For ChatGPT shares, LinkedIn, GPT Showcase: Uses Playwright browser automation
- For regular pages: Uses requests + BeautifulSoup
- Content summaries are passed to AI for better context in analysis
- Falls back gracefully if URL fetching fails

## Query Types

Currently implemented in web UI (chat_analyzer_web.py:30-46):

- **actions** - Tasks, deliverables (outputs: priority 🔴🟡🟢, status ✅📋🔄💬)
- **urls** - Links with context and optional content analysis
- **checkins** - Daily mood scores (outputs: interactive HTML graph with trends)

Additional query types available in CandidateExtractor (not yet in UI):

- **decisions** - Key decisions and agreements
- **meetings** - Schedules, Zoom links, agendas
- **questions** - Questions asked
- **deadlines** - Time-sensitive items
- **assignments** - Direct task assignments

## Deployment Notes

**Railway deployment:**
- Set `OPENROUTER_API_KEY` environment variable (required)
- Optionally set `GOOGLE_TOKEN` for Google Docs/Drive analysis (see GOOGLE_SETUP.md)
- PORT is auto-detected from Railway environment
- Uses gunicorn for production serving
- No persistent storage - all processing is in-memory with temp files cleaned up after serving

**Architecture constraints:**
- No authentication/multi-user support (single public endpoint)
- Single-threaded request handling (one analysis at a time)
- 16MB max file upload size (chat_analyzer_web.py:26)
- Supports .txt and .zip file uploads (extracts .txt from zip)

**Template management:**
- HTML template embedded in chat_analyzer_web.py:356-652 (auto-created if missing)
- Template stored in templates/index.html
- If templates/index.html is deleted, it will be regenerated on app startup

## Cost Considerations

- Uses Claude Haiku 3.5 via OpenRouter (affordable and fast)
- Typical cost: $0.01-0.05 per chat export
- Chunking ensures large files don't exceed token limits
- Can use other models by setting `OPENROUTER_MODEL` environment variable

## Additional Documentation

- **README.md** - User-facing documentation for end users (how to use the app)
- **GOOGLE_SETUP.md** - Detailed setup instructions for Google API integration (OAuth flow, Railway env vars)
