# WhatsApp Chat Analyzer

AI-powered WhatsApp chat analyzer that extracts structured information from chat exports using Claude AI.

## Quick Start

### 1. Set Up Environment

Set the required environment variable:
- `OPENROUTER_API_KEY` - Your OpenRouter API key from https://openrouter.ai/keys

Optional environment variables:
- `OPENROUTER_MODEL` - Model name (default: `anthropic/claude-3.5-haiku`)
- `GOOGLE_TOKEN` - Google OAuth token for analyzing Google Docs/Drive URLs (see GOOGLE_SETUP.md)

### 2. Run the App

**Local development:**
```bash
pip install -r requirements.txt
python3 chat_analyzer_web.py
```

The app will start on http://localhost:8080

**Railway deployment:**
- Set `OPENROUTER_API_KEY` environment variable
- Deploy and the app will auto-configure

### 3. Analyze Your Chats

1. Export your WhatsApp chat:
   - **iPhone**: Open chat → Profile → Export Chat → **Without Media**
   - **Android**: Open chat → ⋮ → More → Export chat → **Without media**
   - This creates a .zip file containing a .txt file

2. Upload the .zip or .txt file to the web interface
3. Choose what to extract:
   - **Actions**: Tasks, assignments, deliverables with priority and status
   - **URLs**: Links shared with context (includes content analysis from links)
   - **Check-ins**: Daily mood scores with interactive graph

4. Set time range (optional):
   - Default: Last 7 days from most recent message
   - Change to any number of days you want to analyze

5. Click "Analyze Chat"
6. View your results in an interactive HTML page

## Features

### Query Types

- **Action Items** - Extract tasks with priority 🔴🟡🟢, status ✅📋🔄💬, assignee, and deadlines
- **URLs & Links** - Capture all shared links with context, importance, and actual content from the URLs
  - Fetches content from Google Docs/Drive (with GOOGLE_TOKEN)
  - Extracts content from ChatGPT shares, LinkedIn profiles, and regular web pages
- **Check-ins** - Visualize daily mood trends with interactive graphs
  - Tracks mood scores over time
  - Shows trends per person with color-coded lines
  - Hover tooltips with full details

### Smart Date Filtering

- Analyzes messages from the last X days (relative to most recent message in file)
- Default: 7 days
- Ensures you get relevant recent data even from older exports

### Interactive HTML Output

- All results displayed as interactive HTML pages
- Priority emojis, status indicators, organized by date
- SVG graphs for check-ins with mood trends over time (hover to see details)
- Opens directly in browser for viewing

## Troubleshooting

### "No results found"

- Make sure your chat export is in the correct format: `[DD/MM/YYYY, HH:MM:SS] Sender: Message`
- Try increasing the time range (e.g., 30 days instead of 7)
- Check that your query type matches the content (e.g., "check-ins" requires messages with mood scores)

### "API Error"

- Verify your `OPENROUTER_API_KEY` environment variable is set correctly
- Check you have API credits remaining at https://openrouter.ai/credits

### File upload issues

- Make sure you're uploading the .zip file from WhatsApp (or the .txt file inside it)
- File must be under 16MB
- Supported formats: .txt and .zip files

### Server won't start

- Check console for error messages
- Make sure all dependencies are installed: `pip install -r requirements.txt`
- For Playwright features, run: `playwright install chromium`

## Technical Details

**Built with:**
- Flask (Python web framework)
- Claude Haiku 3.5 (AI analysis via OpenRouter API)
- Google APIs (optional - for Docs/Drive content analysis)
- Playwright (optional - for ChatGPT/LinkedIn content scraping)
- BeautifulSoup (web page parsing)
- Vanilla JavaScript + SVG (interactive visualizations)

**Cost:**
- Uses Claude Haiku 3.5 via OpenRouter (affordable and fast)
- Typical analysis: $0.01-0.05 per chat export
- Check your usage at https://openrouter.ai/credits
- Can use other models by setting `OPENROUTER_MODEL` environment variable

**Privacy:**
- Chat files processed in memory only
- Temporary files deleted after serving
- Messages sent to OpenRouter API for analysis (see OpenRouter's privacy policy)
- URL content fetched and analyzed for context (optional feature)

## Additional Documentation

- **CLAUDE.md** - Detailed architecture and developer documentation
- **GOOGLE_SETUP.md** - Setup instructions for Google Docs/Drive integration

---

Made with Claude Code 🤖
