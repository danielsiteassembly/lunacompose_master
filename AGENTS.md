# Luna Composer - Agent Documentation

## Overview

Luna Composer is an AI-powered long-form content generation tool integrated within the Supercluster dashboard. It enables users to create thoughtful, data-driven, and hyper-personal content by leveraging Luna's access to comprehensive Visible Light Hub data and OpenAI GPT-4o.

### Purpose

Luna Composer is designed to:
- Generate long-form, thoughtful, and hyper-personal data-driven content
- Provide a professional editing environment for AI-generated content
- Integrate seamlessly with Supercluster dashboard workflows
- Leverage all available VL Hub data (GA4, Lighthouse Insights, ad platforms, competitor analysis, etc.)
- Enable content creation suitable for leadership and executive communication

### Key Differentiators

Unlike Luna Chat (which provides conversational Q&A), Luna Composer:
- **Always generates long-form responses** (minimum 300-500 words for simple queries, 800-1200+ words for complex queries)
- **Focuses on content creation** rather than quick answers
- **Provides a rich text editing environment** with professional formatting tools
- **Auto-saves documents** for 30 days in both WordPress and localStorage
- **Offers export capabilities** (Google Docs, PDF, MP3 audio)
- **Includes feedback mechanisms** (like/dislike) for continuous improvement

---

## Architecture & Integration

### Supercluster Integration

Luna Composer is accessible via the Supercluster dashboard at:
```
https://supercluster.visiblelight.ai/?license=VL-XXXX-XXXX-XXXX/luna/compose/
```

### Page Rendering

When the `/luna/compose/` route is detected:
1. **Three.js animation is disabled** - The Supercluster visualization does not load
2. **Right sidebar is replaced** - The "Omniscient" widget is removed, "Stream Activity" becomes "History"
3. **Controls are removed** - Supercluster navigation controls are hidden
4. **Canvas is removed** - The three.js canvas element is removed from the DOM
5. **Page content container is shown** - The Luna Composer UI is rendered in a full-page container

### License Key Validation

Luna Composer requires:
- A valid VL License Key in the URL parameter
- A `luna_supercluster_shortcode` in the client's VL Hub profile
- The Luna Widget Only plugin installed on the client's WordPress site

### Conditional Loading

The Supercluster loading animation (`supercluster-load-animation.js`) is **NOT loaded** on the Luna Composer page to ensure a clean, focused editing experience.

---

## Features & Functionality

### 1. Canned Prompts & Responses

**Purpose**: Pre-configured prompts that users can click to generate content instantly.

**Implementation**:
- Fetched from WordPress via `/luna_widget/v1/canned-responses` endpoint
- Displayed in a scrollable grid layout
- Each prompt card shows title and excerpt
- Clicking a prompt sends it to Luna API for generation

**User Flow**:
1. User clicks a canned prompt card
2. Prompt is sent to Luna API with `context: 'composer'`
3. Luna generates a long-form, thoughtful response
4. Response is displayed in the editor
5. Document is auto-saved with a unique `document_id`

### 2. Rich Text Editor

**Purpose**: Professional content editing environment.

**Features**:
- **ContentEditable div** - Native browser rich text editing
- **Formatting toolbar** - Bold, italic, underline, lists, etc.
- **Sticky toolbars** - Main toolbar and editor toolbar remain visible during scroll
- **Blur backdrop** - Toolbars have blur effect for visual separation
- **Auto-save** - Saves to WordPress and localStorage every 2 seconds after typing stops

**Editor Toolbar Buttons**:
- **Bold** (`document.execCommand('bold')`)
- **Italic** (`document.execCommand('italic')`)
- **Underline** (`document.execCommand('underline')`)
- **Unordered List** (SVG icon: `list-regular-full.svg`)
- **Ordered List** (SVG icon: `list-ol-solid-full.svg`)

### 3. Document Management

**Auto-Save**:
- **Trigger**: 2-second debounce after user stops typing
- **Storage**: 
  - WordPress (via `/luna_widget/v1/composer/save` endpoint)
  - localStorage (key: `luna_composer_doc_{document_id}`)
  - History list (key: `luna_composer_history_{licenseKey}`)
- **Status Indicator**: Cloud check icon with "Saved to VL Cloud" message (fades after 5 seconds)

**Document ID**:
- Generated on first edit: `doc_{timestamp}_{random}`
- Stored in URL parameter: `?license=VL-XXXX-XXXX-XXXX/luna/compose/&doc={document_id}`
- Used to identify and update existing documents

**History Sidebar**:
- Displays last 30 days of documents
- Fetched from WordPress first, falls back to localStorage
- Each entry shows prompt/title and timestamp
- Clicking an entry loads the document into the editor

### 4. Export & Share

**Export Options**:
- **Google Docs**: Opens document in Google Docs with content pre-filled
- **PDF**: Downloads PDF with exact formatting
- **Audio file (mp3)**: Generates MP3 audio version (currently exports as WebM, requires server-side conversion)

**Share Options**:
- **Copy URL**: Generates unique, shareable URL with document ID
- **Email**: Opens default email client with document content

**Implementation**:
- Export functions use `document.execCommand('copy')` for URL copying
- PDF generation uses browser's print-to-PDF functionality
- Google Docs uses `https://docs.google.com/document/create` with content parameter

### 5. Find & Replace

**Purpose**: Search and replace text within the editor.

**Implementation**:
- Modal dialog with find and replace inputs
- Searches only within the editor's `innerHTML`
- Case-sensitive and whole-word options
- Replaces all occurrences or one at a time

### 6. Dictate & Read Aloud

**Dictate**:
- Uses Web Speech API `SpeechRecognition`
- Converts speech to text and inserts into editor
- Requires browser permission for microphone access

**Read Aloud**:
- Uses Web Speech API `SpeechSynthesis`
- Reads editor content aloud
- Provides stop/pause controls

### 7. Feedback System

**Like/Dislike Buttons**:
- Appear on hover in top-right corner of editor
- Store feedback in WordPress post meta
- Sync to VL Hub via `/vl-hub/v1/composer-feedback` endpoint
- Used to improve Luna's responses over time

**Storage**:
- `feedback_like` / `feedback_dislike` - Count of feedback
- `feedback_like_last` / `feedback_dislike_last` - Timestamp of last feedback
- `feedback_log` - Array of feedback entries (last 100)

### 8. Delete Document

**Purpose**: Permanently delete a document.

**Flow**:
1. User clicks "Move to trash" button
2. Modal appears with warning message
3. User must download PDF backup before deletion is enabled
4. "Delete Permanently" button becomes clickable after backup download
5. Document is deleted from WordPress, localStorage, and history
6. Page reloads with document removed from history

**Safety**:
- Modal prevents accidental deletion
- Requires explicit backup download
- Deletion is permanent (bypasses WordPress trash)

---

## API Endpoints

### 1. Save Document
```
POST /luna_widget/v1/composer/save
```

**Parameters**:
- `license` (string, required) - VL License Key
- `document_id` (string, required) - Unique document identifier
- `prompt` (string, optional) - Original prompt/title
- `content` (string, required) - Document content (HTML)

**Response**:
```json
{
  "success": true,
  "post_id": 123,
  "updated": true
}
```

**Behavior**:
- If document exists (by `document_id`), updates existing post
- If document doesn't exist, creates new post
- Stores as `luna_compose` custom post type

### 2. Fetch Documents
```
GET /luna_widget/v1/composer/fetch
```

**Parameters**:
- `license` (string, required) - VL License Key
- `document_id` (string, optional) - Specific document ID

**Response**:
```json
{
  "documents": [
    {
      "id": "doc_1234567890_abc123",
      "prompt": "Generate a site health report",
      "content": "<p>Document content...</p>",
      "timestamp": 1234567890000,
      "post_id": 123
    }
  ]
}
```

**Behavior**:
- Returns last 30 days of documents if `document_id` not provided
- Returns specific document if `document_id` provided
- Converts timestamps to milliseconds for JavaScript

### 3. Delete Document
```
POST /luna_widget/v1/composer/delete
```

**Parameters**:
- `license` (string, required) - VL License Key
- `document_id` (string, required) - Document ID to delete

**Response**:
```json
{
  "success": true,
  "post_id": 123,
  "deleted": true
}
```

**Behavior**:
- Finds document by `document_id` and `license`
- Permanently deletes post (bypasses trash)

### 4. Feedback
```
POST /luna_widget/v1/composer/feedback
```

**Parameters**:
- `license` (string, required) - VL License Key
- `document_id` (string, required) - Document ID
- `feedback_type` (string, required) - 'like' or 'dislike'
- `prompt` (string, optional) - Original prompt
- `content` (string, optional) - Document content preview

**Response**:
```json
{
  "success": true,
  "post_id": 123,
  "feedback_type": "like",
  "feedback_count": 5
}
```

**Behavior**:
- Creates document if it doesn't exist
- Increments feedback count
- Stores feedback timestamp
- Adds to feedback log
- Syncs to VL Hub

### 5. Chat (Luna API)
```
POST /luna_widget/v1/chat
```

**Parameters**:
- `prompt` (string, required) - User's prompt
- `context` (string, required) - Must be 'composer' for Luna Composer
- `license` (string, optional) - VL License Key

**Response**:
```json
{
  "answer": "Long-form, thoughtful response...",
  "meta": {
    "source": "openai",
    "composer": true,
    "model": "gpt-4o"
  }
}
```

**Special Behavior for Composer**:
- When `context === 'composer'`, Luna **always** generates long-form responses
- Enhanced prompt includes instructions for comprehensive, thoughtful content
- Minimum 300-500 words for simple queries, 800-1200+ words for complex queries
- Uses all available VL Hub data in the response

---

## Data Flow

### Document Creation Flow

1. **User clicks canned prompt or types in editor**
2. **Prompt sent to Luna API** (`/luna_widget/v1/chat` with `context: 'composer'`)
3. **Luna generates response** using:
   - All VL Hub data (GA4, Lighthouse, ads, competitors, etc.)
   - Enhanced system message for long-form content
   - GPT-4o with 4000 max tokens
4. **Response displayed in editor**
5. **Document ID generated** (if not present in URL)
6. **Auto-save triggered** after 2 seconds of inactivity
7. **Saved to WordPress** via `/composer/save` endpoint
8. **Saved to localStorage** for offline access
9. **History list updated** in sidebar

### Document Loading Flow

1. **User clicks history entry**
2. **Fetch from WordPress** via `/composer/fetch?document_id={id}`
3. **If not found, fetch from localStorage**
4. **Load content into editor**
5. **Update URL with document ID**
6. **Cache in localStorage** for faster future access

### Auto-Save Flow

1. **User types in editor**
2. **Input event triggers debounce** (2 seconds)
3. **After 2 seconds of no typing**:
   - Get editor content (`innerHTML`)
   - Get document ID from URL or editor `data-document-id`
   - Create/update document in WordPress
   - Update localStorage
   - Update history list
   - Show "Saved to VL Cloud" message

---

## Technical Implementation

### System Message Enhancement

When `context === 'composer'`, Luna receives an enhanced system message:

```
You are Luna Composer, a sophisticated AI writing assistant that generates 
long-form, thoughtful, and hyper-personal data-driven content. The user is 
using Luna Composer to create editable long-form content, so your response 
must be comprehensive, well-structured, and suitable for professional editing. 
Write in a thoughtful, engaging, and personable manner that feels human and 
authentic, not robotic. Use full sentences, proper paragraph breaks, and 
narrative-style explanations. Include relevant data and insights from the 
provided facts, but weave them naturally into a cohesive narrative. Make the 
content feel personalized and tailored to the user's specific needs. NEVER use 
emoticons, emojis, unicode symbols, or special characters. Use plain text only. 
Generate a substantial, detailed response (minimum 300-500 words for simple 
queries, 800-1200+ words for complex queries) that provides real value and can 
be edited into polished content.
```

### Prompt Enhancement

The user's prompt is enhanced before sending to OpenAI:

```javascript
enhanced_prompt = "You are Luna Composer, a sophisticated AI writing assistant..." + 
                  "\n\nUser request: " + original_prompt;
```

### OpenAI Configuration

For Composer requests:
- **Model**: `gpt-4o`
- **Temperature**: `0.7`
- **Max Tokens**: `4000` (increased from 2000 for comprehensive responses)
- **Facts**: All VL Hub data included in system message

### Custom Post Type

Documents are stored as WordPress custom post type `luna_compose`:

**Post Meta**:
- `prompt` - Original prompt/title
- `answer` - Document content (HTML)
- `document_id` - Unique identifier
- `license` - VL License Key
- `timestamp` - Unix timestamp
- `feedback_like` - Count of likes
- `feedback_dislike` - Count of dislikes
- `feedback_log` - Array of feedback entries

---

## User Interactions

### Initial Load

1. User navigates to `/luna/compose/` in Supercluster
2. Page detects route and calls `renderLunaComposer()`
3. Supercluster elements are removed/hidden
4. Canned prompts are fetched and displayed
5. History sidebar is populated
6. Editor is initialized (empty or with document from URL)

### Using Canned Prompts

1. User clicks a canned prompt card
2. `lunaComposerUsePrompt()` is called
3. Prompt is sent to Luna API
4. Loading state is shown in editor
5. Response is received and displayed
6. Document is auto-saved
7. History is updated

### Manual Editing

1. User types in editor
2. Auto-save debounce timer starts
3. After 2 seconds of inactivity, document is saved
4. "Saved to VL Cloud" message appears
5. Message fades after 5 seconds

### Exporting

1. User clicks "Export to" button
2. Dropdown menu appears
3. User selects format (Google Docs, PDF, MP3)
4. Export function executes
5. File is generated/downloaded

### Sharing

1. User clicks "Share" button
2. Dropdown menu appears
3. User selects "Copy URL" or "Email"
4. URL is copied to clipboard or email client opens

### Deleting

1. User clicks "Move to trash" button
2. Modal appears with warning
3. User must download PDF backup
4. "Delete Permanently" button becomes enabled
5. User confirms deletion
6. Document is deleted from all storage locations
7. Page reloads with updated history

---

## Configuration

### WordPress Settings

**Luna Composer Enabled**:
- Option: `luna_composer_enabled`
- Default: `'1'` (enabled)
- Location: WP Admin > Luna Widget > Composer

**Button Description**:
- Option: `luna_widget_ui_settings['button_desc_compose']`
- Default: "Access Luna Composer to use canned prompts and responses for quick interactions."
- Location: WP Admin > Luna Widget > Settings

### VL Hub Profile

**Required**:
- `luna_supercluster_shortcode` - Shortcode for Luna Chat widget
- Format: `[luna_chat vl_key=VL-XXXX-XXXX-XXXX]`

### Supercluster Route

**URL Pattern**:
```
https://supercluster.visiblelight.ai/?license=VL-XXXX-XXXX-XXXX/luna/compose/
```

**Optional Document Parameter**:
```
?license=VL-XXXX-XXXX-XXXX/luna/compose/&doc=doc_1234567890_abc123
```

---

## Best Practices

### For Content Generation

1. **Use specific prompts** - More specific prompts yield better, more targeted content
2. **Leverage VL Hub data** - Luna has access to all your data; reference specific metrics or services
3. **Request comprehensive reports** - Use phrases like "comprehensive report" or "detailed analysis"
4. **Provide context** - Include background information in your prompts

### For Document Management

1. **Save frequently** - Auto-save handles this, but you can manually trigger by stopping typing
2. **Use descriptive prompts** - The prompt becomes the document title in history
3. **Review before exporting** - Always review generated content before sharing or exporting
4. **Provide feedback** - Use like/dislike buttons to improve future responses

### For Integration

1. **Validate license key** - Always check for valid license before rendering
2. **Handle errors gracefully** - Network errors should not break the UI
3. **Cache locally** - Use localStorage for offline access and faster loading
4. **Sync with WordPress** - Always sync to WordPress for persistence across devices

---

## Troubleshooting

### Document Not Saving

**Symptoms**: "Saved to VL Cloud" message doesn't appear, document not in history

**Solutions**:
1. Check browser console for errors
2. Verify license key is valid
3. Check WordPress REST API is accessible
4. Verify `luna_compose` post type is registered
5. Check localStorage quota (may be full)

### Canned Prompts Not Loading

**Symptoms**: "No canned prompts found" message appears

**Solutions**:
1. Verify canned responses exist in WP Admin > Luna Widget > Canned Responses
2. Check REST API endpoint is accessible
3. Verify license key is correct
4. Check browser console for fetch errors

### Response Too Short

**Symptoms**: Luna generates brief responses instead of long-form content

**Solutions**:
1. Verify `context: 'composer'` is being sent in API request
2. Check system message includes composer instructions
3. Verify `is_composer` flag is set in PHP
4. Check OpenAI max_tokens is set to 4000

### History Not Loading

**Symptoms**: History sidebar shows "No saved documents" but documents exist

**Solutions**:
1. Check WordPress fetch endpoint is working
2. Verify documents are within 30-day window
3. Check localStorage for cached history
4. Verify license key matches document license

### Export Not Working

**Symptoms**: Export buttons don't generate files

**Solutions**:
1. Check browser console for errors
2. Verify editor has content
3. Check browser permissions for downloads
4. For PDF, ensure browser supports print-to-PDF
5. For MP3, verify server-side conversion is implemented

---

## Future Enhancements

### Planned Features

1. **Collaborative Editing** - Multiple users editing same document
2. **Version History** - Track changes and revert to previous versions
3. **Templates** - Pre-built document templates for common use cases
4. **AI Suggestions** - Real-time suggestions while typing
5. **Multi-language Support** - Generate content in multiple languages
6. **Advanced Formatting** - More rich text formatting options
7. **Document Organization** - Folders, tags, and search functionality
8. **Real-time Sync** - Live updates across devices

### Technical Improvements

1. **Server-side MP3 Conversion** - Proper MP3 generation instead of WebM
2. **Offline Mode** - Full functionality without internet connection
3. **Conflict Resolution** - Handle simultaneous edits from multiple devices
4. **Performance Optimization** - Faster loading and smoother editing
5. **Accessibility** - Enhanced screen reader support and keyboard navigation

---

## Related Documentation

- **Luna Chat**: See `luna-widget-only.php` for chat functionality
- **Supercluster Dashboard**: See `index (3).html` for dashboard integration
- **VL Hub API**: See `luna-license-manager.php` for Hub integration
- **WordPress REST API**: See WordPress Codex for REST API documentation

---

## Support

For issues or questions:
1. Check browser console for errors
2. Review WordPress debug logs
3. Verify all dependencies are installed
4. Check VL Hub profile configuration
5. Contact Visible Light support

---

**Last Updated**: 2025-01-XX
**Version**: 1.7.0
**Maintainer**: Visible Light Development Team

