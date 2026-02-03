---
name: add-signal
description: Add Signal as a communication channel (replace WhatsApp, additional channel, control channel, or action-only)
---

# Add Signal Integration

This skill adds Signal capabilities to NanoClaw using signal-cli. It can be configured in four modes:

1. **Replace WhatsApp** - Signal becomes the primary channel
2. **Additional Channel** - Run alongside WhatsApp
3. **Control Channel** - Signal triggers actions, WhatsApp continues
4. **Action-Only** - Used for notifications triggered elsewhere

## Initial Questions

Ask the user:

> How do you want to use Signal with NanoClaw?
>
> **Option 1: Replace WhatsApp**
> - Signal becomes your primary channel
> - All agent interactions go through Signal instead of WhatsApp
>
> **Option 2: Additional Channel**
> - Run both Signal and WhatsApp simultaneously
> - Switch between channels as needed
> - Same agent context across both
>
> **Option 3: Control Channel**
> - Signal is for sending commands and triggering actions
> - WhatsApp remains the primary channel
> - Use Signal when you need quick access
>
> **Option 4: Action-Only**
> - Signal is used only for notifications/alerts
> - No incoming messages (no handler needed)
> - Triggered from scheduled tasks or other channels

Store their choice and proceed to the appropriate section.

---

## Prerequisites (All Modes)

### 1. Install signal-cli

**macOS (Homebrew):**

```bash
brew install signal-cli
```

**Linux (manual install):**

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}.tar.gz"
sudo tar xf "signal-cli-${VERSION}.tar.gz" -C /opt
sudo ln -sf "/opt/signal-cli-${VERSION}/bin/signal-cli" /usr/local/bin/
```

**Requirements:**
- Java Runtime Environment (JRE) 21 or later
- libsignal-client (bundled for x86_64 Linux, Windows, macOS)

Verify installation:

```bash
signal-cli --version
```

### 2. Link to Your Signal Account

Signal requires linking as a secondary device (recommended) or registering a new number. **Linking is strongly recommended** to use your existing Signal account.

Tell the user:

> I'll help you link signal-cli to your existing Signal account. This works like adding Signal Desktop.
>
> **Step 1:** Run this command to generate a linking QR code:

```bash
signal-cli link -n "NanoClaw" | tee /tmp/signal-link.txt | head -1 | xargs qrencode -t UTF8
```

If `qrencode` is not installed:

```bash
# macOS
brew install qrencode

# Linux
sudo apt install qrencode
```

> **Step 2:** On your phone:
> 1. Open Signal
> 2. Go to Settings > Linked Devices
> 3. Tap "Link New Device"
> 4. Scan the QR code displayed in terminal
>
> **Step 3:** Wait for linking to complete (the command will exit automatically)

After linking, sync contacts and groups:

```bash
# Get your account identifier (phone number)
SIGNAL_ACCOUNT=$(signal-cli -o json listAccounts | jq -r '.[0].number')
echo "Your Signal account: $SIGNAL_ACCOUNT"

# Sync contacts and groups
signal-cli -a "$SIGNAL_ACCOUNT" receive
```

### 3. Store Account Identifier

Add your Signal account to `.env`:

```bash
# Get account and add to .env
SIGNAL_ACCOUNT=$(signal-cli -o json listAccounts | jq -r '.[0].number')
echo "SIGNAL_ACCOUNT=\"$SIGNAL_ACCOUNT\"" >> .env
```

Verify:

```bash
grep SIGNAL_ACCOUNT .env
```

### 4. Get Chat Identifiers

Signal uses phone numbers or group IDs for addressing:

**Personal chat ID:** Your contact's phone number in international format (e.g., `+1234567890`)

**Group ID:** List your groups to find IDs:

```bash
signal-cli -a "$SIGNAL_ACCOUNT" listGroups -d
```

Groups are identified by a base64-encoded group ID (e.g., `group.abc123def456...`).

---

## Security Considerations

### End-to-End Encryption

Signal messages are end-to-end encrypted. signal-cli handles the Signal Protocol correctly, but be aware:

- **Keys stored locally**: Your Signal keys are stored in `~/.local/share/signal-cli/data/`
- **Protect this directory**: Anyone with access can read your messages
- **Regular updates required**: signal-cli releases older than 3 months may stop working due to Signal server changes

### Recommendations

```bash
# Set restrictive permissions on signal-cli data
chmod 700 ~/.local/share/signal-cli
chmod 600 ~/.local/share/signal-cli/data/*
```

### Container Security

When running in containers, signal-cli data should be mounted read-only if possible, or with minimal write access:

```typescript
// In container mounts, prefer read-only where possible
const signalDataMount = {
  source: path.join(os.homedir(), '.local/share/signal-cli'),
  target: '/signal-data',
  readonly: false // Required for receiving messages
};
```

---

## Important: Service Conflict (If NanoClaw is Already Running)

If NanoClaw is already running as a background service (e.g., via launchctl), stop it **before testing**:

```bash
# Stop the existing NanoClaw service
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
```

Now you can run `npm run dev` without conflicts.

**After testing is complete**, restart the service:

```bash
# Restart the background service
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
```

---

## Replace WhatsApp Mode

Replace WhatsApp entirely with Signal.

### Step 1: Create Signal Client Module

Create `src/signal-client.ts`:

```typescript
import { spawn, ChildProcess } from 'child_process';
import { EventEmitter } from 'events';
import { logger } from './logger.js';

interface SignalMessage {
  envelope: {
    source: string;
    sourceNumber?: string;
    sourceName?: string;
    timestamp: number;
    dataMessage?: {
      message: string;
      groupInfo?: {
        groupId: string;
        type: string;
      };
    };
  };
}

export class SignalClient extends EventEmitter {
  private daemon: ChildProcess | null = null;
  private account: string;
  private buffer: string = '';

  constructor(account: string) {
    super();
    this.account = account;
  }

  async start(): Promise<void> {
    return new Promise((resolve, reject) => {
      // Start signal-cli in JSON-RPC mode
      this.daemon = spawn('signal-cli', [
        '-a', this.account,
        'jsonRpc'
      ], {
        stdio: ['pipe', 'pipe', 'pipe']
      });

      this.daemon.stdout?.on('data', (data: Buffer) => {
        this.buffer += data.toString();
        this.processBuffer();
      });

      this.daemon.stderr?.on('data', (data: Buffer) => {
        logger.debug({ stderr: data.toString() }, 'signal-cli stderr');
      });

      this.daemon.on('error', (err) => {
        logger.error({ err }, 'signal-cli daemon error');
        reject(err);
      });

      this.daemon.on('exit', (code) => {
        logger.info({ code }, 'signal-cli daemon exited');
        this.daemon = null;
      });

      // Subscribe to receive messages
      this.sendCommand('subscribeReceive', {});

      // Give daemon time to initialize
      setTimeout(() => resolve(), 1000);
    });
  }

  private processBuffer(): void {
    const lines = this.buffer.split('\n');
    this.buffer = lines.pop() || '';

    for (const line of lines) {
      if (!line.trim()) continue;
      try {
        const json = JSON.parse(line);
        this.handleResponse(json);
      } catch (err) {
        logger.debug({ line }, 'Non-JSON output from signal-cli');
      }
    }
  }

  private handleResponse(response: any): void {
    if (response.method === 'receive') {
      // Incoming message notification
      const envelope = response.params?.envelope;
      if (envelope?.dataMessage?.message) {
        this.emit('message', {
          envelope: {
            source: envelope.sourceNumber || envelope.source,
            sourceNumber: envelope.sourceNumber,
            sourceName: envelope.sourceName || envelope.sourceNumber,
            timestamp: envelope.timestamp,
            dataMessage: envelope.dataMessage
          }
        } as SignalMessage);
      }
    } else if (response.id !== undefined) {
      // Response to a command we sent
      this.emit(`response:${response.id}`, response);
    }
  }

  private requestId = 0;

  private sendCommand(method: string, params: any): Promise<any> {
    return new Promise((resolve, reject) => {
      const id = ++this.requestId;
      const command = JSON.stringify({
        jsonrpc: '2.0',
        id,
        method,
        params
      }) + '\n';

      this.once(`response:${id}`, (response) => {
        if (response.error) {
          reject(new Error(response.error.message));
        } else {
          resolve(response.result);
        }
      });

      if (!this.daemon?.stdin?.write(command)) {
        reject(new Error('Failed to write to signal-cli'));
      }
    });
  }

  async sendMessage(recipient: string, message: string): Promise<void> {
    const isGroup = recipient.startsWith('group.');

    await this.sendCommand('send', {
      ...(isGroup ? { groupId: recipient } : { recipient: [recipient] }),
      message
    });

    logger.info({ recipient, length: message.length }, 'Signal message sent');
  }

  async setTyping(recipient: string, isTyping: boolean): Promise<void> {
    // Signal supports typing indicators but they're less common in CLI usage
    // This is a no-op for now but can be implemented if needed
  }

  stop(): void {
    if (this.daemon) {
      this.daemon.kill();
      this.daemon = null;
    }
  }
}
```

### Step 2: Add Signal Handler to index.ts

At the top of `src/index.ts`, add the import:

```typescript
import { SignalClient } from './signal-client.js';
```

Create the client instance after your logger setup:

```typescript
const signalClient = new SignalClient(process.env.SIGNAL_ACCOUNT!);
```

Add the message handler:

```typescript
// Signal message handler
signalClient.on('message', async (msg) => {
  const envelope = msg.envelope;
  const content = envelope.dataMessage?.message;
  if (!content) return;

  const isGroup = !!envelope.dataMessage?.groupInfo;
  const chatId = isGroup
    ? `group.${envelope.dataMessage.groupInfo.groupId}`
    : envelope.source;

  const senderId = envelope.source;
  const senderName = envelope.sourceName || envelope.source;

  logger.info(
    { chatId, isGroup, senderName },
    `Signal message: ${content}`
  );

  try {
    // Process message through existing routing
    await processIncomingMessage({
      chatId,
      sender: senderId,
      senderName,
      content,
      timestamp: envelope.timestamp,
      isGroup,
      platform: 'signal'
    });
  } catch (error) {
    logger.error({ error, chatId }, 'Error processing Signal message');
    await signalClient.sendMessage(chatId, 'Sorry, something went wrong.');
  }
});
```

Add helper functions:

```typescript
async function sendSignalMessage(chatId: string, text: string): Promise<void> {
  try {
    await signalClient.sendMessage(chatId, text);
  } catch (error) {
    logger.error({ error, chatId }, 'Failed to send Signal message');
    throw error;
  }
}
```

### Step 3: Start Signal Client

Find where services start their connection. Add:

```typescript
// Start Signal client
try {
  await signalClient.start();
  logger.info('Signal client started');

  process.once('SIGINT', () => {
    logger.info('Shutting down Signal client');
    signalClient.stop();
  });
  process.once('SIGTERM', () => {
    logger.info('Shutting down Signal client');
    signalClient.stop();
  });
} catch (error) {
  logger.error({ error }, 'Failed to start Signal client');
  process.exit(1);
}
```

### Step 4: Remove WhatsApp Code

Find and comment out or delete:
- `connectWhatsApp()` function calls
- WhatsApp-specific message handlers
- WhatsApp initialization code

Keep the existing `processIncomingMessage` or agent routing function.

### Step 5: Update Group Memory

Update `groups/main/CLAUDE.md`:

```markdown
## Communication

You are accessed via Signal. Users send you messages in their chat or group.

Your Signal account: ${SIGNAL_ACCOUNT}
```

### Step 6: Test

Run the service:

```bash
npm run dev
```

Test by sending a message via Signal. Verify:
- Bot responds
- No errors in logs
- Messages are encrypted end-to-end

---

## Additional Channel Mode

Run Signal and WhatsApp simultaneously.

### Step 1: Add Signal Handler

Follow **Replace WhatsApp Mode -> Steps 1-2** to add the Signal client and handler.

Do NOT remove any WhatsApp code.

### Step 2: Wire Both Channels

Both handlers should call the same `processIncomingMessage` function:

```typescript
// WhatsApp handler calls:
await processIncomingMessage({ platform: 'whatsapp', ... });

// Signal handler calls:
await processIncomingMessage({ platform: 'signal', ... });
```

The routing should use `chat_jid` to store the platform and chat ID:
- WhatsApp: `whatsapp:1234567890@s.whatsapp.net`
- Signal: `signal:+1234567890` or `signal:group.abc123...`

### Step 3: Update Send Functions

Route replies to the correct platform:

```typescript
async function sendMessage(
  chatJid: string,
  text: string
): Promise<void> {
  if (chatJid.startsWith('signal:')) {
    const chatId = chatJid.replace('signal:', '');
    await sendSignalMessage(chatId, text);
  } else {
    // WhatsApp: include ASSISTANT_NAME prefix
    const message = `${ASSISTANT_NAME}: ${text}`;
    await sendWhatsAppMessage(chatJid, message);
  }
}
```

### Step 4: Update Memory

Update `groups/main/CLAUDE.md`:

```markdown
## Communication

You are accessed via Signal and WhatsApp. Users can reach you on either platform.

### Signal
- Account: ${SIGNAL_ACCOUNT}

### WhatsApp
- Phone: [USER_PHONE]
```

### Step 5: Test Both Channels

```bash
npm run dev
```

Test by:
1. Sending a message on WhatsApp - verify it works
2. Sending a message on Signal - verify it works
3. Check that context is shared

---

## Control Channel Mode

Signal triggers actions, WhatsApp remains primary.

### Step 1: Add Minimal Signal Handler

Create `src/signal-client.ts` as in Replace WhatsApp mode.

Add to `src/index.ts`:

```typescript
import { SignalClient } from './signal-client.js';

const signalClient = new SignalClient(process.env.SIGNAL_ACCOUNT!);

signalClient.on('message', async (msg) => {
  const envelope = msg.envelope;
  const content = envelope.dataMessage?.message;
  if (!content) return;

  const chatId = envelope.dataMessage?.groupInfo
    ? `group.${envelope.dataMessage.groupInfo.groupId}`
    : envelope.source;

  logger.info({ chatId, content }, 'Signal control message');

  try {
    const mainGroup = getRegisteredGroup('main');

    if (!mainGroup) {
      await signalClient.sendMessage(chatId, 'Agent not configured');
      return;
    }

    const response = await runContainerAgent(mainGroup, {
      prompt: content,
      chatJid: `signal:${chatId}`,
      isScheduledTask: false
    });

    if (response.status === 'success') {
      await signalClient.sendMessage(chatId, response.result);
    } else {
      await signalClient.sendMessage(chatId, 'Error processing command');
    }
  } catch (error) {
    logger.error({ error, chatId }, 'Error in Signal control handler');
    await signalClient.sendMessage(chatId, 'An error occurred');
  }
});
```

### Step 2: Start Signal Client

Add client startup (same as Replace WhatsApp mode).

### Step 3: Keep WhatsApp Running

Do NOT remove WhatsApp code.

### Step 4: Update Memory

```markdown
## Communication

Primary channel: WhatsApp
Control channel: Signal (send commands to trigger actions)
```

### Step 5: Test

Send a command through Signal - it should trigger the agent and reply on Signal.

---

## Action-Only Mode

Signal used only for notifications.

### Step 1: Create Minimal Send-Only Client

Create `src/signal-sender.ts`:

```typescript
import { execSync } from 'child_process';
import { logger } from './logger.js';

const SIGNAL_ACCOUNT = process.env.SIGNAL_ACCOUNT!;

export async function sendSignalNotification(
  recipient: string,
  message: string
): Promise<void> {
  try {
    const isGroup = recipient.startsWith('group.');
    const args = isGroup
      ? ['-a', SIGNAL_ACCOUNT, 'send', '-g', recipient, '-m', message]
      : ['-a', SIGNAL_ACCOUNT, 'send', recipient, '-m', message];

    execSync(`signal-cli ${args.map(a => `'${a}'`).join(' ')}`, {
      stdio: 'pipe',
      timeout: 30000
    });

    logger.info({ recipient }, 'Signal notification sent');
  } catch (error) {
    logger.error({ error, recipient }, 'Failed to send Signal notification');
    throw error;
  }
}
```

### Step 2: Use in Notifications

```typescript
import { sendSignalNotification } from './signal-sender.js';

// In your task scheduler or notification code:
await sendSignalNotification(
  process.env.SIGNAL_NOTIFICATION_RECIPIENT!,
  'Task completed successfully'
);
```

### Step 3: Update Memory

```markdown
## Notifications

Notifications are sent to Signal recipient: ${SIGNAL_NOTIFICATION_RECIPIENT}
```

---

## Privacy Model: How Registered Chats Work

### Signal vs Telegram/WhatsApp

Signal's privacy model differs from other platforms:

- **No bot discovery**: Unlike Telegram, there's no public bot directory
- **Contact-based**: You can only message contacts who have your number
- **Group access**: You must be added to groups by existing members

### Registered Chats Format

```json
{
  "signal:+1234567890": {
    "name": "personal",
    "folder": "main",
    "trigger": "@Sara",
    "added_at": "2026-02-02T14:30:00.000Z"
  },
  "signal:group.abc123...": {
    "name": "team_group",
    "folder": "team",
    "trigger": "@Sara",
    "added_at": "2026-02-02T15:45:00.000Z"
  }
}
```

### Security Implications

- **Registered chats**: Full two-way communication with your agent
- **Unregistered chats**: Silently ignored
- **E2E encryption**: All messages are encrypted; signal-cli handles the Signal Protocol
- **No metadata leakage**: Signal minimizes metadata compared to other platforms

---

## Group Permissions

In Signal groups, the bot receives all messages if added as a member. Unlike Telegram, there's no "privacy mode" to configure.

**Important**: Your Signal account (the one linked to signal-cli) will appear as a regular group member. Other members can see you're in the group.

---

## Rate Limits

Signal has informal rate limits to prevent spam:

- **No official documentation**: Unlike Telegram, limits aren't published
- **Practical limit**: ~10-20 messages per minute to different recipients
- **Same conversation**: No practical limit for back-and-forth

If you send too many messages too fast, Signal may temporarily restrict your account.

```typescript
// Add delays for batch notifications
for (const recipient of recipients) {
  await sendSignalNotification(recipient, 'Alert');
  await new Promise(r => setTimeout(r, 500)); // 500ms between messages
}
```

---

## Chat ID Formats

| Type | Format | Example |
|------|--------|---------|
| Personal | Phone number | `+1234567890` |
| Group | Base64 group ID | `group.abc123def456...` |

Store all chat IDs as **strings** in your database.

---

## Testing Procedure

### For Replace WhatsApp or Additional Channel:

1. **Start the service:**
   ```bash
   npm run dev
   ```

2. **Send a test message via Signal:**
   - Open Signal on your phone
   - Message your linked account (or have someone else message you)

3. **Verify response:**
   - Should get a reply
   - No errors in logs

4. **Test groups (if applicable):**
   - Create or use existing group
   - Send a message mentioning the trigger
   - Bot should respond

5. **Monitor logs:**
   ```bash
   tail -f logs/nanoclaw.log | grep -i signal
   ```

### For Control Channel:

1. **Verify WhatsApp works first**
2. **Send a message via Signal**
3. **Check that the main group receives input and responds back on Signal**

### For Action-Only:

1. **Trigger a scheduled task**
2. **Verify Signal message arrives**
3. **Check logs for send confirmation**

---

## Known Issues & Fixes

### "No registered account" error

**Cause**: signal-cli not linked to an account

**Fix**:
```bash
# Check linked accounts
signal-cli listAccounts

# If empty, re-link
signal-cli link -n "NanoClaw"
```

### Messages not received

**Cause**: signal-cli daemon not running or not subscribed

**Fix**:
```bash
# Test receiving manually
signal-cli -a "$SIGNAL_ACCOUNT" receive

# Check if daemon is running
ps aux | grep signal-cli
```

### "Untrusted identity" error

**Cause**: Contact's safety number changed

**Fix**:
```bash
# Trust new identity
signal-cli -a "$SIGNAL_ACCOUNT" trust -a CONTACT_NUMBER
```

### Java not found

**Cause**: JRE not installed or not in PATH

**Fix**:
```bash
# macOS
brew install openjdk

# Linux
sudo apt install default-jre

# Verify
java -version
```

### signal-cli version too old

**Cause**: Signal server requires newer client

**Fix**:
```bash
# Update signal-cli
brew upgrade signal-cli

# Or manually download latest
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
echo "Latest version: $VERSION"
```

---

## Troubleshooting Commands

### Check signal-cli status:
```bash
signal-cli listAccounts
signal-cli -a "$SIGNAL_ACCOUNT" listGroups
```

### Test sending a message:
```bash
signal-cli -a "$SIGNAL_ACCOUNT" send -m "Test message" +1234567890
```

### Receive pending messages:
```bash
signal-cli -a "$SIGNAL_ACCOUNT" receive
```

### Check signal-cli data:
```bash
ls -la ~/.local/share/signal-cli/data/
```

### View container logs:
```bash
cat groups/main/logs/container-*.log | tail -50
```

---

## Removing Signal Integration

To remove Signal entirely:

1. **Remove from `src/index.ts`:**
   - Delete `import { SignalClient } from './signal-client.js'`
   - Delete client initialization
   - Delete message handler
   - Delete `sendSignalMessage` and related functions
   - Delete client startup code

2. **Remove signal-client.ts:**
   ```bash
   rm src/signal-client.ts
   ```

3. **Remove from `.env`:**
   ```bash
   # Remove this line:
   # SIGNAL_ACCOUNT="..."
   ```

4. **Update memory files:**
   - Remove Signal section from `groups/*/CLAUDE.md`

5. **Optionally unlink from Signal:**
   ```bash
   # On your phone: Settings > Linked Devices > Remove "NanoClaw"
   ```

6. **Rebuild:**
   ```bash
   npm run build
   ```

---

## Additional Resources

- [signal-cli GitHub](https://github.com/AsamK/signal-cli)
- [signal-cli Wiki](https://github.com/AsamK/signal-cli/wiki)
- [Signal Protocol](https://signal.org/docs/)
- [Linking Devices](https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning))
- [JSON-RPC Service](https://github.com/AsamK/signal-cli/wiki/JSON-RPC-service)
