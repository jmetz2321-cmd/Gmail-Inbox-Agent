# Mentorship Inbox Agent — Code

Google Apps Script that automatically monitors your Gmail for MentorCruise and Leland emails, classifies them, and generates Claude-powered draft replies every 15 minutes.

---

```javascript
/**
 * ════════════════════════════════════════════════════════════════
 *  MENTORSHIP INBOX AGENT — Google Apps Script
 *  Auto-drafts replies for MentorCruise & Leland emails
 *
 *  SETUP (one-time):
 *    1. Go to https://script.google.com → New Project
 *    2. Paste this entire file, replacing any existing code
 *    3. Add your Anthropic API key: click the ⚙️ icon (Project Settings
 *       → Script Properties) and add key: ANTHROPIC_API_KEY
 *    4. Run setupTrigger() once from the editor (Run menu)
 *    5. Approve the Gmail + Script permissions when prompted
 *
 *  After setup, the script runs every 15 minutes automatically.
 *  It only drafts replies for emails it hasn't seen before.
 * ════════════════════════════════════════════════════════════════
 */

// ── Configuration ────────────────────────────────────────────────

var CONFIG = {
  SEARCH_QUERY: "from:mentorcruise.com OR from:leland.com OR from:lelandapp.com",
  CHECK_INTERVAL_MINUTES: 15,
  PROCESSED_KEY: "processedMessageIds",   // stored in Script Properties
  MAX_PER_RUN: 20,                         // max emails to process per trigger run
  MODEL: "claude-sonnet-4-20250514",
  SIGN_OFF: "Jon",                         // your name for sign-offs
};


// ── Entry points ─────────────────────────────────────────────────

/**
 * Run this ONCE manually to install the recurring trigger.
 */
function setupTrigger() {
  // Remove any existing triggers for this function first
  ScriptApp.getProjectTriggers().forEach(function(t) {
    if (t.getHandlerFunction() === "checkNewEmails") {
      ScriptApp.deleteTrigger(t);
    }
  });

  ScriptApp.newTrigger("checkNewEmails")
    .timeBased()
    .everyMinutes(CONFIG.CHECK_INTERVAL_MINUTES)
    .create();

  Logger.log("✅ Trigger installed — checkNewEmails will run every " + CONFIG.CHECK_INTERVAL_MINUTES + " minutes.");

  // Run immediately on first setup so you see it work right away
  checkNewEmails();
}

/**
 * Main function — called by the trigger every 15 minutes.
 * Finds new mentorship emails, classifies them, drafts replies.
 */
function checkNewEmails() {
  var apiKey = PropertiesService.getScriptProperties().getProperty("ANTHROPIC_API_KEY");
  if (!apiKey) {
    Logger.log("❌ ANTHROPIC_API_KEY not set. Go to Project Settings → Script Properties to add it.");
    return;
  }

  var processed = getProcessedIds();
  var threads = GmailApp.search(CONFIG.SEARCH_QUERY, 0, 50);
  var drafted = 0;

  for (var i = 0; i < threads.length && drafted < CONFIG.MAX_PER_RUN; i++) {
    var thread = threads[i];
    var messages = thread.getMessages();

    // Only process the latest message in the thread
    var msg = messages[messages.length - 1];
    var msgId = msg.getId();

    // Skip already-processed messages
    if (processed[msgId]) continue;

    // Classify email
    var subject = msg.getSubject();
    var body = msg.getPlainBody().substring(0, 1500); // trim for API
    var from = msg.getFrom();
    var category = classifyEmail(subject, body, from);

    if (!category) {
      // Not a relevant category — mark as seen and skip
      markProcessed(processed, msgId);
      continue;
    }

    Logger.log("📧 [" + category.toUpperCase() + "] " + subject + " — from: " + from);

    // Generate draft reply via Claude API
    var draftBody = generateDraft(apiKey, category, from, subject, body);

    if (draftBody) {
      // Create as a draft reply in the same thread
      thread.createDraftReply(draftBody);

      Logger.log("  ✅ Draft created for: " + subject);
      drafted++;
    }

    markProcessed(processed, msgId);
    saveProcessedIds(processed);

    // Small pause to avoid rate-limiting the Anthropic API
    Utilities.sleep(1000);
  }

  Logger.log("Run complete. " + drafted + " new drafts created.");
}


// ── Classification ────────────────────────────────────────────────

function classifyEmail(subject, body, from) {
  var s = (subject + " " + body + " " + from).toLowerCase();

  // Stops — check first (cancellations can look like meetings)
  if (/cancel|cancell|stop mentorship|pause mentorship|end.*mentor|mentor.*end|no longer|opt.out|withdraw|quit mentorship/.test(s)) {
    return "stops";
  }

  // Reviews
  if (/review|rating|rated|left.*review|new review|wrote.*review|gave.*feedback/.test(s)) {
    return "reviews";
  }

  // Meetings & Leads (new applications + scheduled calls)
  if (
    /scheduled|rescheduled|new event|calendar invite|meeting.*confirm|confirm.*meeting|mentorcal|session.*booked|booked.*session|call.*scheduled|scheduled.*call|applied to you|has applied|new application|unreviewed application|awaiting your response|intro.*call|intro.*meet|want to meet|free.*intro/.test(s) &&
    !/message from|unread message/.test(s)
  ) {
    return "meetings";
  }

  // Messages (mentee chat messages)
  if (/message from|unread message|you have a new message|new message from|reminder.*message/.test(s)) {
    return "messages";
  }

  return null; // Not a relevant mentorship email
}


// ── Draft generation via Claude API ──────────────────────────────

function generateDraft(apiKey, category, from, subject, body) {
  var systemPrompts = {
    meetings: (
      "You are a friendly, professional mentor named " + CONFIG.SIGN_OFF + ". " +
      "If this is a new mentee application, write a warm 3-5 sentence reply expressing interest and proposing next steps. " +
      "If it's a meeting confirmation or scheduling email, write a brief acknowledgment (2-3 sentences). " +
      "If it's asking for intro call feedback, confirm it went well and you're happy to proceed. " +
      "Sign off as " + CONFIG.SIGN_OFF + ". Write ONLY the email body, no subject line."
    ),
    messages: (
      "You are a helpful, encouraging mentor named " + CONFIG.SIGN_OFF + ". " +
      "Write a thoughtful, warm reply to your mentee's message. " +
      "Be conversational, supportive, and specific to what they said. Keep it concise (3-6 sentences). " +
      "Sign off as " + CONFIG.SIGN_OFF + ". Write ONLY the email body, no subject line."
    ),
    reviews: (
      "You are a grateful mentor named " + CONFIG.SIGN_OFF + ". " +
      "Write a short, warm thank-you for the review (2-3 sentences). " +
      "Sign off as " + CONFIG.SIGN_OFF + ". Write ONLY the email body, no subject line."
    ),
    stops: (
      "You are an empathetic, understanding mentor named " + CONFIG.SIGN_OFF + ". " +
      "Write a kind, gracious reply acknowledging the mentee's decision to pause or end the mentorship. " +
      "Wish them well and leave the door open. Keep it to 3-4 sentences. " +
      "Sign off as " + CONFIG.SIGN_OFF + ". Write ONLY the email body, no subject line."
    ),
  };

  var systemPrompt = systemPrompts[category] || systemPrompts["messages"];

  var userMessage = (
    "Draft a reply to this email:\n" +
    "From: " + from + "\n" +
    "Subject: " + subject + "\n\n" +
    "Email content:\n" + body
  );

  var payload = JSON.stringify({
    model: CONFIG.MODEL,
    max_tokens: 400,
    system: systemPrompt,
    messages: [{ role: "user", content: userMessage }],
  });

  var options = {
    method: "post",
    contentType: "application/json",
    headers: {
      "x-api-key": apiKey,
      "anthropic-version": "2023-06-01",
    },
    payload: payload,
    muteHttpExceptions: true,
  };

  try {
    var response = UrlFetchApp.fetch("https://api.anthropic.com/v1/messages", options);
    var data = JSON.parse(response.getContentText());

    if (data.content && data.content[0] && data.content[0].text) {
      return data.content[0].text.trim();
    } else {
      Logger.log("  ⚠️ Unexpected API response: " + JSON.stringify(data).substring(0, 200));
      return null;
    }
  } catch (e) {
    Logger.log("  ❌ API error: " + e.message);
    return null;
  }
}


// ── Processed ID tracking ─────────────────────────────────────────

function getProcessedIds() {
  var raw = PropertiesService.getScriptProperties().getProperty(CONFIG.PROCESSED_KEY);
  return raw ? JSON.parse(raw) : {};
}

function markProcessed(processed, msgId) {
  processed[msgId] = Date.now();
  // Trim to last 2000 entries to avoid hitting the 9KB property limit
  var keys = Object.keys(processed);
  if (keys.length > 2000) {
    var sorted = keys.sort(function(a, b) { return processed[a] - processed[b]; });
    sorted.slice(0, keys.length - 2000).forEach(function(k) { delete processed[k]; });
  }
}

function saveProcessedIds(processed) {
  PropertiesService.getScriptProperties().setProperty(CONFIG.PROCESSED_KEY, JSON.stringify(processed));
}


// ── Utility functions ─────────────────────────────────────────────

/**
 * Run this if you want the script to re-process all emails from scratch.
 */
function resetProcessedIds() {
  PropertiesService.getScriptProperties().deleteProperty(CONFIG.PROCESSED_KEY);
  Logger.log("✅ Processed IDs cleared. Next run will process all matching emails.");
}

/**
 * Run this to see the current trigger status.
 */
function checkTriggerStatus() {
  var triggers = ScriptApp.getProjectTriggers();
  if (triggers.length === 0) {
    Logger.log("No triggers installed. Run setupTrigger() to install.");
  } else {
    triggers.forEach(function(t) {
      Logger.log("Trigger: " + t.getHandlerFunction() + " | Type: " + t.getTriggerSource());
    });
  }
}
```
