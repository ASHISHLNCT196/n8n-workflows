# Social Media Video Agent

An n8n workflow that writes, produces and publishes short-form video posts for you on **LinkedIn, Instagram Reels, a Facebook Page and YouTube Shorts**. Nothing goes live until you tap **✅ Publish** in Telegram.

File: `Social_Media_Video_Agent.json` (import it with *Workflows → Import from file*).

## What it does

```
 Google Sheet queue (daily 9:00) ─┐
                                  ├─► Claude writes the script + captions for each platform, in your voice
 Telegram message to your bot ────┘                │
                                                   ▼
                     Video source:  your Telegram upload │ your Google Drive file │ HeyGen AI avatar video
                                                   ▼
                     Telegram preview (video + all captions) ─► ✅ Publish / ❌ Discard (expires after 24 h)
                                                   ▼
             LinkedIn (Videos + Posts API) · Instagram Reel · Facebook Page video · YouTube Short
                                                   ▼
                          Telegram message with a link for each post (or the error)
```

### Giving it work

**From Telegram** (only your chat ID is accepted; messages from anyone else are ignored):

| You send | What happens |
|---|---|
| `Reel about the 3 mistakes I made hiring my first engineer` | AI avatar video + captions on all default platforms |
| A video (≤ 20 MB) with a caption describing it | Your video is used; captions are written around it |
| `https://drive.google.com/file/d/<id>/view launch day recap` | Your Drive video is used (any size) |
| …plus a last line `to: linkedin, youtube` | Only those platforms (`li`, `ig`, `fb`, `yt` also work) |

**From the Google Sheet:** use `topic_queue_template.csv` as the `Queue` tab.

| Column | Meaning |
|---|---|
| Topic | What the post is about |
| Platforms | Optional, e.g. `linkedin, instagram`. Empty means `defaultPlatforms` |
| Video | Optional Google Drive link or file ID. Empty means an AI video is generated |
| Notes | Optional extra instructions for the writer |
| Status | Set to `Queued`. The agent changes it to `Drafting`, then `Published` or `Rejected` |
| Published At | Filled in by the agent |

Each morning the agent takes one `Queued` row. Change the **Daily Queue Check** trigger to post more or less often.

## Setup

### 1. Agent Config node
Fill in every `YOUR_…` value:

| Field | Where to get it |
|---|---|
| `ownerChatId` | Message your bot, then open `https://api.telegram.org/bot<TOKEN>/getUpdates` and copy `message.chat.id` |
| `sheetId` | The long ID in the Google Sheet URL |
| `brandVoice` | A few sentences on who you are, your audience and your tone. This matters most for quality |
| `heygenAvatarId` / `heygenVoiceId` | HeyGen → create a digital twin of yourself (Instant Avatar) and a voice clone, then copy their IDs (or use `GET https://api.heygen.com/v2/avatars` and `/v2/voices`) |
| `linkedinAuthorUrn` | `urn:li:person:<sub>`, where `sub` comes from `GET https://api.linkedin.com/v2/userinfo`. For a company page use `urn:li:organization:<id>` (needs `w_organization_social`) |
| `linkedinApiVersion` | A current `YYYYMM` LinkedIn API version. Versions are retired after about a year, so update it if LinkedIn returns a version error |
| `igUserId` | The Instagram Business/Creator account ID linked to your Facebook Page |
| `fbPageId` | Your Facebook Page ID |
| `metaApiVersion` | Graph API version, e.g. `v23.0` |
| `youtubePrivacy` | `public`, `unlisted` or `private` |

Also put your chat ID in the **Alert Me On Error** node. That node runs in a separate error execution and can't read Agent Config.

### 2. Credentials

| Credential (n8n type) | Used by | Notes |
|---|---|---|
| Telegram API | All Telegram nodes | Bot token from @BotFather |
| Anthropic API | Claude Model | The writer. You can switch to another chat model node if you prefer |
| Google Sheets OAuth2 API | Queue nodes | |
| Google Drive OAuth2 API | Download Drive Video | |
| Header Auth, name `X-Api-Key` | HeyGen nodes | Value = HeyGen API key |
| Header Auth, name `Authorization` | Instagram + Facebook nodes | Value = `OAuth <long-lived Page access token>` with `instagram_basic`, `instagram_content_publish`, `pages_manage_posts`, `pages_read_engagement`, `publish_video` |
| LinkedIn OAuth2 API | LinkedIn nodes | Scope `w_member_social` (the "Share on LinkedIn" product) |
| YouTube OAuth2 API | YouTube: Upload Short | |

### 3. Finish
1. **Workflow settings → Error workflow:** choose this workflow so that unexpected errors reach you on Telegram.
2. n8n must be reachable over public HTTPS (`WEBHOOK_URL`), for the Telegram trigger and the approval buttons.
3. Activate the workflow and send your bot a test message.

## Reliability built in
- **Approval required:** a Telegram preview with Publish/Discard buttons. It expires after 24 h and counts as discarded.
- **Owner only:** chats from other Telegram users are dropped.
- **Polling loops with limits:** HeyGen rendering (~30 min max), LinkedIn video processing and Instagram container processing each give up after a fixed number of checks and tell you why.
- **Platforms fail independently:** if one platform rejects the upload, you get a Telegram message naming the failing step and the API error, and the other platforms still publish.
- **Platform formatting:** LinkedIn reserved characters are escaped, YouTube titles are kept under 100 characters with `#Shorts` added, `<`/`>` are removed for YouTube, and the Instagram caption is capped at 2,200 characters.

## Video requirements
- Vertical 9:16 MP4 (H.264/AAC), 3–90 s, so it qualifies as both a Reel and a Short.
- Telegram uploads are limited to 20 MB by the Bot API; use a Drive link for larger files.
- The Telegram preview is skipped for files over 50 MB; you still get the approval message.

## Ideas to extend
- Send `freeText` approval instead of buttons so you can reply with edits and regenerate.
- Add X/TikTok publishers by copying a platform branch.
- Log every post with its links to a second sheet tab and pull weekly analytics.
