<div align="center">

**An extended Baileys fork — albums, interactive messages, newsletters, and more.**

[![License: MIT](https://img.shields.io/badge/license-MIT-blue?style=flat-square&labelColor=0d1117)](LICENSE)
[![Node.js](https://img.shields.io/badge/node-%3E%3D20-339933?style=flat-square&labelColor=0d1117)](https://nodejs.org)
[![ESM + CJS](https://img.shields.io/badge/ESM_%2B_CJS-supported-f7df1e?style=flat-square&labelColor=0d1117)](https://nodejs.org)
[![Based on Baileys](https://img.shields.io/badge/based_on-Baileys-white?style=flat-square&labelColor=0d1117)](https://github.com/WhiskeySockets/Baileys)

</div>

---

> **wb-core** is an independent fork maintained by [nulswa](https://github.com/nulswa). It is not affiliated with, endorsed by, or officially connected to WhatsApp Inc. or Meta Platforms. nulswa holds no responsibility for misuse of this library, including any derivative forks. Use responsibly and in accordance with WhatsApp's Terms of Service.
>
> wb-core does not claim to be a replacement for [WhiskeySockets/Baileys](https://github.com/WhiskeySockets/Baileys). It is a targeted extension built on top of it.

---

## Table of Contents

- [What wb-core adds](#what-wb-core-adds)
- [Ecosystem](#ecosystem)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Auth State](#auth-state)
- [Data Store](#data-store)
- [WhatsApp ID Reference](#whatsapp-id-reference)
- [Sending Messages](#sending-messages)
  - [Text](#text)
  - [Mention](#mention)
  - [Reaction](#reaction)
  - [Pin](#pin)
  - [Forward](#forward)
  - [Contact](#contact)
  - [Location](#location)
  - [Event](#event)
  - [Group Invite](#group-invite)
  - [Product](#product)
  - [Poll](#poll)
  - [Button Response](#button-response)
  - [Rich Response](#rich-response)
  - [Code Block](#code-block)
  - [Inline Entities](#inline-entities)
  - [Table](#table)
  - [Status Mention](#status-mention)
- [Media Messages](#media-messages)
  - [Image](#image)
  - [Video](#video)
  - [Sticker](#sticker)
  - [Audio](#audio)
  - [Document](#document)
  - [Album](#album)
  - [Sticker Pack](#sticker-pack)
- [Interactive Messages](#interactive-messages)
  - [Buttons](#buttons)
  - [List](#list)
  - [Native Flow & Carousel](#native-flow--carousel)
  - [Hydrated Template](#hydrated-template)
- [Payment Messages](#payment-messages)
- [Message Options](#message-options)
- [Modifying Messages](#modifying-messages)
- [Additional APIs](#additional-apis)
  - [Find User ID](#find-user-id)
  - [Custom Pairing Code](#custom-pairing-code)
  - [Image Processing](#image-processing)
  - [Newsletter Management](#newsletter-management)
  - [Group Management](#group-management)
  - [Profile Management](#profile-management)
  - [Privacy Management](#privacy-management)
  - [Events Reference](#events-reference)
- [Credits](#credits)

---

## What wb-core adds

wb-core is built on [WhiskeySockets/Baileys](https://github.com/WhiskeySockets/Baileys) and extends it without breaking the upstream API surface. Everything Baileys supports, wb-core also supports.

**Internal fixes & adjustments**

| Fix | Detail |
|---|---|
| Newsletter media upload | Resolves upstream failure when sending media to newsletter channels |
| `makeInMemoryStore` restored | Reintroduced with ESM compatibility and Baileys v7 adjustments |
| FFmpeg: `exec` → `spawn` | Safer child process handling |
| `@napi-rs/image` backend | Added as a supported image processing option alongside `sharp` and `jimp` |

**Extended message types**

- Album (mixed image + video)
- Interactive messages: buttons, lists, native flows, templates, carousels
- Rich response (AI-style formatted output with code blocks and tables)
- Code block messages (with built-in tokenizer)
- Table messages
- Status mention messages
- Sticker pack messages
- Group status messages
- Payment messages: requests, invites, orders, invoices

**Additional options per message**

| Flag | Effect |
|---|---|
| `ai` | Adds AI icon (DM only) |
| `mentionAll` | Mentions all group participants without supplying JIDs manually |
| `ephemeral` | Wraps message in `ephemeralMessage` |
| `viewOnce` / `viewOnceV2` / `viewOnceV2Extension` | Wraps in respective view-once containers |
| `groupStatus` | Sends as group status (groups only) |
| `secureMetaServiceLabel` | Applies secure meta service label |
| `interactiveAsTemplate` | Wraps interactive message as template |
| `raw` | Passes a raw proto structure directly — do not use for exploitation |
| `externalAdReply` | Ad thumbnail without manual `contextInfo` construction |

---

## Ecosystem

wb-core is part of a small set of Baileys forks maintained under [nulswa](https://github.com/nulswa):

| Library | Best for | Repo |
|---|---|---|
| **wb-norm** | Simple bots, drop-in Baileys replacement with `@lid` fix | [nulswa/wb-norm](https://github.com/nulswa/wb-norm) |
| **wb-core** | Extended message types, newsletters, interactives | This repo |

**Looking for a ready-to-run bot base?**

[**nulswa/toru-bot**](https://github.com/nulswa/toru-bot) is a lightweight, forkable WhatsApp bot starter compatible with both `wb-norm` and `wb-core`. Clone it, fork it, or use it as-is for simple bots or full chatbot deployments.

> **Note:** toru-bot is **not compatible** with `wb-optima`. That variant is designed exclusively for `toru-optima`.

---

## Installation

wb-core is installed directly from GitHub — no npm package required.

```bash
# via npm (package.json)
npm install github:nulswa/wb-core

# or pin to a specific ref
npm install github:nulswa/wb-core#main
```

Or add it to `package.json` manually:

```json
{
  "dependencies": {
    "wb-core": "github:nulswa/wb-core"
  }
}
```

#### Import

```javascript
// ESM
import { makeWASocket } from 'wb-core'

// CommonJS (Node.js 20+)
const { makeWASocket } = require('wb-core')
```

---

## Quick Start

A minimal connection with pairing code authentication:

```javascript
import {
  makeWASocket,
  delay,
  DisconnectReason,
  useMultiFileAuthState
} from 'wb-core'
import { Boom } from '@hapi/boom'
import pino from 'pino'

const MY_PHONE_NUMBER = '15551234567' // E.164, no + prefix
const logger = pino({ level: 'silent' })

async function connect() {
  const { state, saveCreds } = await useMultiFileAuthState('./auth')

  const sock = makeWASocket({ logger, auth: state })

  sock.ev.on('creds.update', saveCreds)

  // ── Connection lifecycle ──────────────────────────────────────────────────
  sock.ev.on('connection.update', async (update) => {
    const { connection, lastDisconnect } = update

    if (connection === 'connecting' && !sock.authState.creds.registered) {
      await delay(1500)
      const code = await sock.requestPairingCode(MY_PHONE_NUMBER)
      console.log('Pairing code:', code)
    }

    if (connection === 'close') {
      const code = new Boom(lastDisconnect?.error)?.output?.statusCode
      const retry  = code !== DisconnectReason.loggedOut
      console.log('Connection closed —', lastDisconnect?.error?.message)
      if (retry) connect()
    }

    if (connection === 'open') {
      console.log('✓ Connected to WhatsApp')
    }
  })

  // ── Incoming messages ─────────────────────────────────────────────────────
  sock.ev.on('messages.upsert', async ({ messages }) => {
    for (const m of messages) {
      if (!m.message) continue
      console.log('↓', m.key.remoteJid)

      await sock.sendMessage(m.key.remoteJid, { text: '👋 Hello!' })
    }
  })
}

connect()
```

---

## Auth State

```javascript
const { state, saveCreds } = await useMultiFileAuthState('./auth')
```

`useMultiFileAuthState` writes credentials to a local directory. For production, implement your own auth state backed by a database — Signal keys must be read and written atomically.

> You can also use `useSingleFileAuthState` as an alternative. It includes an internal caching mechanism, so there is no need to wrap `state.keys` with `makeCacheableSignalKeyStore`.

---

## Data Store

wb-core ships with `makeInMemoryStore` for development use. It is not recommended for production — storing full chat history in RAM is expensive at scale.

```javascript
import {
  makeWASocket,
  makeInMemoryStore,
  delay,
  DisconnectReason,
  useMultiFileAuthState
} from 'wb-core'
import { Boom } from '@hapi/boom'
import pino from 'pino'

const STORE_PATH = './store.json'
const logger = pino({ level: 'silent' })

async function connect() {
  const { state, saveCreds } = await useMultiFileAuthState('./auth')
  const sock = makeWASocket({ logger, auth: state })

  const store = makeInMemoryStore({ logger, socket: sock })
  store.bind(sock.ev)
  store.readFromFile(STORE_PATH)

  // Persist store every 3 minutes
  setInterval(() => store.writeToFile(STORE_PATH), 180_000)

  sock.ev.on('creds.update', saveCreds)

  sock.ev.on('chats.upsert', () => {
    console.log('Chats:', store.chats.all())
  })

  sock.ev.on('contacts.upsert', () => {
    console.log('Contacts:', Object.values(store.contacts))
  })

  // ... connection.update, messages.upsert, etc.
}

connect()
```

---

## WhatsApp ID Reference

| Entity | Format | Example |
|---|---|---|
| User (PN) | `[country][number]@s.whatsapp.net` | `15551234567@s.whatsapp.net` |
| User (LID) | `[id]@lid` | `43411111111111@lid` |
| Group | `[ts]-[random]@g.us` | `1234567890-987654@g.us` |
| Newsletter | `[id]@newsletter` | `1231111111111@newsletter` |
| Meta AI | `11111111111@bot` | — |
| Broadcast list | `[ts]@broadcast` | — |
| Stories | `status@broadcast` | — |

---

## Sending Messages

All messages use the same interface:

```javascript
await sock.sendMessage(jid, content, options)
// options: { quoted, ephemeralExpiration, ... }
```

---

### Text

```javascript
// Plain text
await sock.sendMessage(jid, { text: 'Hello!' }, { quoted: message })

// With basic link preview
const url = 'https://github.com/nulswa/wb-core'
await sock.sendMessage(jid, {
  text: url + ' — check it out!',
  linkPreview: {
    'matched-text': url,
    title: 'wb-core',
    description: 'Extended Baileys fork by nulswa',
    previewType: 0,
    jpegThumbnail: fs.readFileSync('./thumb.jpg')
  }
})

// With large link preview and favicon
import { prepareWAMessageMedia } from 'wb-core'

const { imageMessage: image } = await prepareWAMessageMedia(
  { image: { url: './thumb.jpg' } },
  { upload: sock.waUploadToServer, mediaTypeOverride: 'thumbnail-link' }
)
image.height = 720
image.width  = 480

await sock.sendMessage(jid, {
  text: url + ' — check it out!',
  linkPreview: {
    'matched-text': url,
    title: 'wb-core',
    description: 'Extended Baileys fork by nulswa',
    previewType: 0,
    jpegThumbnail: fs.readFileSync('./thumb.jpg'),
    highQualityThumbnail: image,
    linkPreviewMetadata: {
      linkMediaDuration: 0,
      socialMediaPostType: 1
    }
  },
  favicon: { url: './favicon.jpg' }
})
```

---

### Mention

```javascript
// Single mention
await sock.sendMessage(jid, {
  text: 'Hey @15551234567!',
  mentions: ['15551234567@s.whatsapp.net']
}, { quoted: message })

// Mention everyone (no JIDs required)
await sock.sendMessage(jid, {
  text: '👋 Hello @all',
  mentionAll: true
}, { quoted: message })
```

---

### Reaction

```javascript
await sock.sendMessage(jid, {
  react: { key: message.key, text: '🔥' }
})
// Empty string removes the reaction
```

---

### Pin

```javascript
// Duration in seconds: 86400 (1d), 604800 (7d), 2592000 (30d)
await sock.sendMessage(jid, {
  pin: message.key,
  time: 604800,
  type: 1   // 0 = unpin
})
```

---

### Forward

```javascript
await sock.sendMessage(jid, {
  forward: message,
  force: true   // optional — forces forward even if already forwarded
})
```

---

### Contact

```javascript
const vcard = [
  'BEGIN:VCARD',
  'VERSION:3.0',
  'FN:Ada Lovelace',
  'ORG:Engineering;',
  'TEL;type=CELL;type=VOICE;waid=15551234567:+1 555 123 4567',
  'END:VCARD'
].join('\n')

await sock.sendMessage(jid, {
  contacts: {
    displayName: 'Ada Lovelace',
    contacts: [{ vcard }]
  }
}, { quoted: message })
```

---

### Location

```javascript
await sock.sendMessage(jid, {
  location: {
    degreesLatitude: 37.7749,
    degreesLongitude: -122.4194,
    name: 'San Francisco'
  }
}, { quoted: message })
```

---

### Event

```javascript
await sock.sendMessage(jid, {
  event: {
    name: 'Team Sync',
    description: 'Weekly engineering sync.',
    call: 'video',              // or 'audio', optional
    startDate: new Date(Date.now() + 3_600_000),
    endDate:   new Date(Date.now() + 7_200_000),
    isCancelled: false,
    isScheduleCall: false,
    extraGuestsAllowed: false,
    location: {
      name: 'Remote',
      degreesLatitude: 0,
      degreesLongitude: 0
    }
  }
}, { quoted: message })
```

---

### Group Invite

```javascript
const inviteCode = groupUrl.split('chat.whatsapp.com/')[1]?.split('?')[0]

await sock.sendMessage(jid, {
  groupInvite: {
    inviteCode,
    inviteExpiration: Date.now() + 86_400_000,
    text: 'You are invited to join our group.',
    jid: '1234567890-987654@g.us',
    subject: 'Engineering Team'
  }
}, { quoted: message })
```

---

### Product

```javascript
import { randomUUID } from 'crypto'

await sock.sendMessage(jid, {
  image: { url: './product.jpg' },
  body: 'Check out this product!',
  footer: 'wb-core store',
  product: {
    currencyCode: 'USD',
    description: 'A great product.',
    priceAmount1000: 29_000,
    salePriceAmount1000: 24_000,
    productId: randomUUID(),
    productImageCount: 1,
    signedUrl: 'https://github.com/nulswa/wb-core',
    title: 'My Product',
    url: 'https://github.com/nulswa/wb-core'
  },
  businessOwnerJid: '0@s.whatsapp.net'
})
```

---

### Poll

```javascript
// Standard poll
await sock.sendMessage(jid, {
  poll: {
    name: 'Preferred stack?',
    values: ['Node.js', 'Bun', 'Deno'],
    selectableCount: 1,
    toAnnouncementGroup: false
  }
}, { quoted: message })

// Quiz (newsletters only)
await sock.sendMessage('1231111111111@newsletter', {
  poll: {
    name: 'What is 2 + 2?',
    values: ['3', '4', '5'],
    correctAnswer: '4',
    pollType: 1
  }
})

// Poll result
await sock.sendMessage(jid, {
  pollResult: {
    name: 'Results: Preferred stack?',
    votes: [
      { name: 'Node.js', voteCount: 12 },
      { name: 'Bun',     voteCount: 8 },
      { name: 'Deno',    voteCount: 3 }
    ],
    pollType: 0
  }
})
```

---

### Button Response

```javascript
// buttonsResponseMessage
await sock.sendMessage(jid, {
  type: 'plain',
  buttonReply: { id: '#Menu', displayText: 'Main Menu' }
}, { quoted: message })

// interactiveResponseMessage
await sock.sendMessage(jid, {
  flowReply: {
    format: 0,
    text: 'Response',
    name: 'menu_options',
    paramsJson: JSON.stringify({ id: '#Menu', description: 'Main Menu' })
  }
}, { quoted: message })

// listResponseMessage
await sock.sendMessage(jid, {
  listReply: { title: 'Selection', description: 'Main Menu', id: '#Menu' }
}, { quoted: message })

// templateButtonReplyMessage
await sock.sendMessage(jid, {
  type: 'template',
  buttonReply: { id: '#Menu', displayText: 'Main Menu', index: 1 }
}, { quoted: message })
```

---

### Rich Response

Renders AI-style formatted output with code blocks, tables, and prose sections.

```javascript
import { tokenizeCode } from 'wb-core'

const language = 'javascript'
const code = 'console.log("Hello, World!")'

await sock.sendMessage(jid, {
  richResponse: [
    { text: 'Example Usage' },
    { language, code: tokenizeCode(code, language) },
    { text: 'Simple, right?\n' },
    { text: 'Runtime Comparison' },
    {
      title: 'Node.js vs Bun vs Deno',
      table: [
        { isHeading: true,  items: ['',            'Node.js',   'Bun',              'Deno']    },
        { isHeading: false, items: ['Engine',      'V8 (C++)', 'JavaScriptCore',   'V8 (C++)'] },
        { isHeading: false, items: ['Performance', '4/5',       '5/5',              '4/5']     }
      ]
    },
    { text: 'Does this help?' }
  ]
})
```

---

### Code Block

Built-in tokenizer is applied automatically.

```javascript
await sock.sendMessage(jid, {
  headerText: '## Example',
  contentText: '---',
  code: 'console.log("Hello, World!")',
  language: 'javascript',
  footerText: 'Simple, right?'
})
```

---

### Inline Entities

```javascript
await sock.sendMessage(jid, {
  headerText: '## Useful Links',
  contentText: '---',
  links: [
    { text: '1. GitHub',   title: 'Source hosting',      url: 'https://github.com' },
    { text: '2. wb-core',  title: 'Extended Baileys fork', url: 'https://github.com/nulswa/wb-core' }
  ],
  footerText: '---'
})
```

---

### Table

```javascript
await sock.sendMessage(jid, {
  headerText: '## Runtime Comparison',
  contentText: '---',
  title: 'Node.js vs Bun vs Deno',
  table: [
    ['',            'Node.js',  'Bun',             'Deno'   ],
    ['Engine',      'V8 (C++)', 'JavaScriptCore',  'V8 (C++)'],
    ['Performance', '4/5',      '5/5',             '4/5'    ]
  ],
  noHeading: false,
  footerText: 'First row is treated as the header.'
})
```

---

### Status Mention

Send a message that appears as a status mention to multiple recipients:

```javascript
await sock.sendMessage([jidA, jidB, jidC], { text: 'Hello! 👋' })
```

---

## Media Messages

For all media types, the source can be a `Buffer`, `{ url: string }` (local path or HTTP/S URL), or `{ stream: Readable }`. Prefer URL or stream — wb-core streams and encrypts without loading the full asset into memory.

---

### Image

```javascript
await sock.sendMessage(jid, {
  image: { url: './photo.jpg' },
  caption: 'Caption here.'
}, { quoted: message })
```

---

### Video

```javascript
await sock.sendMessage(jid, {
  video: { url: './clip.mp4' },
  caption: 'Caption here.',
  gifPlayback: false,   // true = looping GIF-style
  ptv: false            // true = video note (circle)
}, { quoted: message })
```

---

### Sticker

```javascript
await sock.sendMessage(jid, {
  sticker: { url: './sticker.webp' }
}, { quoted: message })
```

---

### Audio

For voice notes, encode first:
```bash
ffmpeg -i input.mp4 -avoid_negative_ts make_zero -ac 1 -codec:a libopus output.ogg
```

```javascript
await sock.sendMessage(jid, {
  audio: { url: './voice.ogg' },
  ptt: true   // true = voice note, false = audio player
}, { quoted: message })
```

---

### Document

```javascript
await sock.sendMessage(jid, {
  document: { url: './report.pdf' },
  mimetype: 'application/pdf',
  caption: 'Q3 Report'
}, { quoted: message })
```

---

### Album

Mixed image and video in a single swipeable album:

```javascript
await sock.sendMessage(jid, {
  album: [
    { image: { url: './photo1.jpg' }, caption: 'First photo'  },
    { video: { url: './clip1.mp4'  }, caption: 'First clip'   },
    { image: { url: './photo2.jpg' }, caption: 'Second photo' },
    { video: { url: './clip2.mp4'  }, caption: 'Second clip'  }
  ]
}, { quoted: message })
```

---

### Sticker Pack

> If `sharp` or `@napi-rs/image` is not installed, `cover` and `stickers` must already be in WebP format.

```javascript
await sock.sendMessage(jid, {
  cover: { url: './cover.webp' },
  stickers: [
    { data: { url: './s1.webp' } },
    { data: { url: './s2.webp' } },
    { data: { url: './s3.webp' } }
  ],
  name: 'My Sticker Pack',
  publisher: 'nulswa',
  description: 'wb-core'
}, { quoted: message })
```

---

## Interactive Messages

### Buttons

```javascript
// Simple buttons
await sock.sendMessage(jid, {
  text: 'Choose an option:',
  footer: 'wb-core',
  buttons: [
    { text: 'Option A', id: '#OptionA' },
    { text: 'Option B', id: '#OptionB' }
  ]
}, { quoted: message })

// Buttons with media and native flow sections
await sock.sendMessage(jid, {
  image: { url: './banner.jpg' },
  caption: 'Choose:',
  footer: 'wb-core',
  buttons: [
    { text: 'Rate', id: '#Rate' },
    {
      text: 'Browse',
      sections: [
        {
          title: 'Category A',
          rows: [{ header: '', title: 'Item 1', description: '', id: '#Item1' }]
        },
        {
          title: 'Category B',
          highlight_label: 'Popular',
          rows: [{ header: '', title: 'Item 2', description: '', id: '#Item2' }]
        }
      ]
    }
  ]
}, { quoted: message })
```

---

### List

> Only works in private chat (`@s.whatsapp.net`).

```javascript
await sock.sendMessage(jid, {
  text: 'Pick from the list:',
  footer: 'wb-core',
  buttonText: 'Open List',
  title: 'Menu',
  sections: [
    {
      title: 'Section 1',
      rows: [{ title: 'AI Tools', description: '', rowId: '#AI' }]
    },
    {
      title: 'Section 2',
      rows: [{ title: 'Search', description: '', rowId: '#Search' }]
    }
  ]
}, { quoted: message })
```

---

### Native Flow & Carousel

```javascript
// Native Flow
await sock.sendMessage(jid, {
  image: { url: './banner.jpg' },
  caption: 'Interactive options:',
  footer: 'wb-core',
  optionText: 'Select',
  optionTitle: 'Options',
  offerText: 'Limited offer!',
  offerCode: 'SAVE20',
  offerUrl: 'https://github.com/nulswa/wb-core',
  offerExpiration: Date.now() + 3_600_000,
  nativeFlow: [
    { text: 'Greet',  id: '#Greet', icon: 'review' },
    { text: 'Call',   call: '15551234567'            },
    { text: 'Copy',   copy: 'nulswa/wb-core'         },
    { text: 'Open',   url: 'https://github.com/nulswa/wb-core', useWebview: true },
    {
      text: 'Browse',
      sections: [
        {
          title: 'Section 1',
          rows: [{ header: '', title: 'Item 1', description: '', id: '#Item1' }]
        }
      ],
      icon: 'default'
    }
  ],
  interactiveAsTemplate: false
}, { quoted: message })

// Carousel with per-card native flows
await sock.sendMessage(jid, {
  text: 'Browse our collection:',
  footer: 'wb-core',
  cards: [
    {
      image: { url: './card1.jpg' },
      caption: 'Card 1',
      footer: 'wb-core',
      nativeFlow: [{ text: 'View', url: 'https://github.com/nulswa/wb-core', useWebview: true }]
    },
    {
      image: { url: './card2.jpg' },
      caption: 'Card 2',
      footer: 'wb-core',
      offerText: 'New offer!',
      offerCode: 'SAVE10',
      offerUrl: 'https://github.com/nulswa/wb-core',
      offerExpiration: Date.now() + 3_600_000,
      nativeFlow: [{ text: 'View', url: 'https://github.com/nulswa/wb-core' }]
    }
  ]
}, { quoted: message })
```

---

### Hydrated Template

```javascript
await sock.sendMessage(jid, {
  title: 'Hello',
  image: { url: './banner.jpg' },
  caption: 'Template message:',
  footer: 'wb-core',
  templateButtons: [
    { text: 'Order now',   id: '#Order'                                     },
    { text: 'View source', url: 'https://github.com/nulswa/wb-core'         },
    { text: 'Call us',     call: '15551234567'                               }
  ]
}, { quoted: message })
```

---

## Payment Messages

```javascript
// Payment invite
await sock.sendMessage(jid, { paymentInviteServiceType: 3 }) // 1, 2, or 3

// Invoice (limited support)
await sock.sendMessage(jid, {
  image: { url: './invoice.jpg' },
  invoiceNote: 'Invoice #1042'
})

// Order
await sock.sendMessage(jid, {
  orderText: 'Your order',
  thumbnail: fs.readFileSync('./thumb.jpg')  // must be Buffer
}, { quoted: message })

// Request payment
await sock.sendMessage(jid, {
  text: 'Please complete your payment.',
  requestPaymentFrom: '0@s.whatsapp.net'
})
```

---

## Message Options

These flags can be added to any compatible message payload:

```javascript
// AI icon (DM only)
await sock.sendMessage(jid, { image: { url: './img.jpg' }, caption: 'AI-generated', ai: true })

// Ephemeral wrapper
await sock.sendMessage(jid, { image: { url: './img.jpg' }, ephemeral: true })

// External ad reply (ad thumbnail without manual contextInfo)
await sock.sendMessage(jid, {
  text: 'Check this out.',
  externalAdReply: {
    title: 'wb-core',
    body: 'Extended Baileys fork',
    thumbnail: fs.readFileSync('./thumb.jpg'),
    largeThumbnail: false,
    url: 'https://github.com/nulswa/wb-core'
  }
}, { quoted: message })

// Group status (groups only)
await sock.sendMessage(jid, { image: { url: './img.jpg' }, groupStatus: true })

// Secure meta service label
await sock.sendMessage(jid, { text: 'Labeled message.', secureMetaServiceLabel: true })

// View once variants
await sock.sendMessage(jid, { image: { url: './img.jpg' }, viewOnce: true })
await sock.sendMessage(jid, { image: { url: './img.jpg' }, viewOnceV2: true })
await sock.sendMessage(jid, { image: { url: './img.jpg' }, viewOnceV2Extension: true })

// Raw proto (advanced — do not use for exploitation)
await sock.sendMessage(jid, {
  extendedTextMessage: {
    text: 'Built from raw proto.',
    contextInfo: {
      externalAdReply: {
        title: 'wb-core',
        thumbnail: fs.readFileSync('./thumb.jpg'),
        sourceApp: 'whatsapp',
        showAdAttribution: true,
        mediaType: 1
      }
    }
  },
  raw: true
}, { quoted: message })
```

---

## Modifying Messages

```javascript
// Delete for everyone
await sock.sendMessage(jid, { delete: message.key })

// Edit text
await sock.sendMessage(jid, { text: 'Corrected text.', edit: message.key })

// Edit media caption
await sock.sendMessage(jid, { caption: 'Updated caption.', edit: message.key })
```

---

## Additional APIs

### Find User ID

Resolves between phone number (PN) and local identifier (LID):

```javascript
// From phone number
const ids = await sock.findUserId('15551234567@s.whatsapp.net')
// → { phoneNumber: '15551234567@s.whatsapp.net', lid: '43411111111111@lid' }

// From LID
const ids = await sock.findUserId('43411111111111@lid')
// → same shape; lid: 'id-not-found' if unresolvable
```

---

### Custom Pairing Code

```javascript
const code = await sock.requestPairingCode('15551234567', 'WBCORE01')
console.log('Pairing code:', code) // → 'WBCORE01'
```

---

### Image Processing

wb-core auto-selects the available image processing backend: `sharp` → `@napi-rs/image` → `jimp`.

```javascript
import { getImageProcessingLibrary } from 'wb-core'
import { readFile } from 'fs/promises'

const lib = await getImageProcessingLibrary()
const input = './photo.jpg'
const width = 512
let output

if (lib.sharp?.default) {
  output = await lib.sharp.default(input)
    .resize(width)
    .jpeg({ quality: 80 })
    .toBuffer()
}
else if (lib.image?.Transformer) {
  const buf = Buffer.isBuffer(input) ? input : await readFile(input)
  output = await new lib.image.Transformer(buf)
    .resize(width, undefined, 0)
    .jpeg(50)
}
else if (lib.jimp?.Jimp) {
  const img = await lib.jimp.Jimp.read(input)
  output = await img
    .resize({ w: width, mode: lib.jimp.ResizeStrategy.BILINEAR })
    .getBuffer('image/jpeg', { quality: 50 })
}
else {
  throw new Error('No image processing library available. Install sharp, @napi-rs/image, or jimp.')
}
```

---

### Newsletter Management

```javascript
// Create
await sock.newsletterCreate('My Channel', 'Weekly updates.')

// Metadata & subscribers
const meta = await sock.newsletterMetadata('1231111111111@newsletter')
const subs = await sock.newsletterSubscribers('1231111111111@newsletter')

// Follow / unfollow
await sock.newsletterFollow('1231111111111@newsletter')
await sock.newsletterUnfollow('1231111111111@newsletter')

// Mute / unmute
await sock.newsletterMute('1231111111111@newsletter')
await sock.newsletterUnmute('1231111111111@newsletter')

// Admin management
await sock.newsletterDemote('1231111111111@newsletter', '15551234567@s.whatsapp.net')
await sock.newsletterChangeOwner('1231111111111@newsletter', '15551234567@s.whatsapp.net')

// Update details
await sock.newsletterUpdateName('1231111111111@newsletter', 'Updated Name')
await sock.newsletterUpdateDescription('1231111111111@newsletter', 'Updated description.')
await sock.newsletterUpdatePicture('1231111111111@newsletter', { url: './photo.jpg' })
await sock.newsletterRemovePicture('1231111111111@newsletter')

// React to a message in the newsletter
await sock.newsletterReactMessage('1231111111111@newsletter', '100', '💛')

// Fetch messages
const messages = await sock.newsletterFetchMessages('jid', '1231111111111@newsletter', 50, 0, 0)

// List subscribed newsletters
const newsletters = await sock.newsletterSubscribed()

// Delete newsletter
await sock.newsletterDelete('1231111111111@newsletter')
```

---

### Group Management

```javascript
// Create
const group = await sock.groupCreate('My Group', ['15551234567@s.whatsapp.net'])

// Metadata
const meta = await sock.groupMetadata(jid)

// Invite management
const code = await sock.groupInviteCode(jid)
await sock.groupRevokeInvite(jid)
await sock.groupAcceptInvite(code)
const info = await sock.groupGetInviteInfo('https://chat.whatsapp.com/ABC123')

// Participants — actions: 'add' | 'remove' | 'promote' | 'demote'
await sock.groupParticipantsUpdate(jid, ['15551234567@s.whatsapp.net'], 'add')

// Join requests
const pending = await sock.groupRequestParticipantsList(jid)
await sock.groupRequestParticipantsUpdate(jid, ['15551234567@s.whatsapp.net'], 'approve')
// or 'reject'

// Info updates
await sock.groupUpdateSubject(jid, 'New Name')
await sock.groupUpdateDescription(jid, 'New description.')
await sock.updateProfilePicture(jid, { url: './photo.jpg' })
await sock.removeProfilePicture(jid)

// Settings
await sock.groupSettingUpdate(jid, 'announcement')      // admins only chat
await sock.groupSettingUpdate(jid, 'not_announcement')  // all members chat
await sock.groupSettingUpdate(jid, 'locked')            // admins only edit info
await sock.groupSettingUpdate(jid, 'unlocked')          // all members edit info
await sock.groupMemberAddMode(jid, 'admin_add')         // admins only add
await sock.groupMemberAddMode(jid, 'all_member_add')    // all members add
await sock.groupJoinApprovalMode(jid, 'on')             // approval required
await sock.groupJoinApprovalMode(jid, 'off')

// Ephemeral messages (seconds: 86400, 604800, 7776000, or 0 to disable)
await sock.groupToggleEphemeral(jid, 604800)

// Leave & fetch all
await sock.groupLeave(jid)
const all = await sock.groupFetchAllParticipating()

// Member label
await sock.updateMemberLabel(jid, 'my-label')
```

---

### Profile Management

```javascript
// Profile picture
const url = await sock.profilePictureUrl(jid, 'image')
await sock.updateProfilePicture(jid, { url: './photo.jpg' })
await sock.removeProfilePicture(jid)

// Name & status
await sock.updateProfileName('My Bot')
await sock.updateProfileStatus('Available.')

// Presence
await sock.sendPresenceUpdate('available', jid)
await sock.presenceSubscribe(jid)

// Read receipts
await sock.readMessages([message.key])
await sock.sendReceipt(jid, participant, [messageId], 'read')

// Block / unblock
await sock.updateBlockStatus(jid, 'block')
await sock.updateBlockStatus(jid, 'unblock')
const blocked = await sock.fetchBlocklist()

// Contacts
await sock.addOrEditContact(jid, { displayName: 'Alice' })
await sock.removeContact(jid)

// Labels
await sock.addChatLabel(jid, labelId)
await sock.removeChatLabel(jid, labelId)
await sock.addMessageLabel(jid, messageId, labelId)

// Chat modifications
await sock.chatModify({ archive: true, lastMessageOrig: message, lastMessage: message }, jid)
await sock.star(jid, [{ id: messageId, fromMe: true }], true)

// App state sync
await sock.resyncAppState(['regular', 'critical_block'], true)

// Business profile
const biz = await sock.getBusinessProfile(jid)
```

---

### Privacy Management

```javascript
// Last seen: 'all' | 'contacts' | 'contact_blacklist' | 'nobody'
await sock.updateLastSeenPrivacy('contacts')

// Online: 'all' | 'match_last_seen'
await sock.updateOnlinePrivacy('match_last_seen')

// Profile picture
await sock.updateProfilePicturePrivacy('contacts')

// Status
await sock.updateStatusPrivacy('contacts')

// Read receipts: 'all' | 'none'
await sock.updateReadReceiptsPrivacy('none')

// Groups add: 'all' | 'contacts' | 'contact_blacklist'
await sock.updateGroupsAddPrivacy('contacts')

// Messages: 'all' | 'contacts' | 'nobody'
await sock.updateMessagesPrivacy('contacts')

// Calls: 'everyone'
await sock.updateCallPrivacy('everyone')

// Default disappearing mode (seconds: 86400, 604800, 7776000, 0)
await sock.updateDefaultDisappearingMode(604800)

// Link previews
await sock.updateDisableLinkPreviewsPrivacy(true)
```

---

### Events Reference

```javascript
sock.ev.on('connection.update',              (e) => {})
sock.ev.on('creds.update',                   (e) => {})
sock.ev.on('messaging-history.set',          (e) => {})
sock.ev.on('messaging-history.status',       (e) => {})
sock.ev.on('chats.upsert',                   (e) => {})
sock.ev.on('chats.update',                   (e) => {})
sock.ev.on('chats.delete',                   (e) => {})
sock.ev.on('chats.lock',                     (e) => {})
sock.ev.on('lid-mapping.update',             (e) => {})
sock.ev.on('presence.update',                (e) => {})
sock.ev.on('contacts.upsert',                (e) => {})
sock.ev.on('contacts.update',                (e) => {})
sock.ev.on('messages.delete',                (e) => {})
sock.ev.on('messages.update',                (e) => {})
sock.ev.on('messages.media-update',          (e) => {})
sock.ev.on('messages.upsert',                (e) => {})
sock.ev.on('messages.reaction',              (e) => {})
sock.ev.on('message-receipt.update',         (e) => {})
sock.ev.on('groups.upsert',                  (e) => {})
sock.ev.on('groups.update',                  (e) => {})
sock.ev.on('group-participants.update',      (e) => {})
sock.ev.on('group.join-request',             (e) => {})
sock.ev.on('group.member-tag.update',        (e) => {})
sock.ev.on('blocklist.set',                  (e) => {})
sock.ev.on('blocklist.update',               (e) => {})
sock.ev.on('call',                           (e) => {})
sock.ev.on('labels.edit',                    (e) => {})
sock.ev.on('labels.association',             (e) => {})
sock.ev.on('newsletter.reaction',            (e) => {})
sock.ev.on('newsletter.view',                (e) => {})
sock.ev.on('newsletter-participants.update', (e) => {})
sock.ev.on('newsletter-settings.update',     (e) => {})
sock.ev.on('settings.update',                (e) => {})
```

---

## Credits

wb-core is built on [WhiskeySockets/Baileys](https://github.com/WhiskeySockets/Baileys). Full credit to the original Baileys authors and contributors:

- [adiwajshing](https://github.com/adiwajshing) — original author
- [purpshell](https://github.com/purpshell) — primary upstream maintainer
- [jlucaso1](https://github.com/jlucaso1) — upstream contributor

Protocol Buffer definitions provided by [WPPConnect/wa-proto](https://github.com/wppconnect-team/wa-proto).

> Modification, removal, or misrepresentation of these credits is strictly prohibited. Any redistribution or fork must preserve this section in its original form.

---

<div align="center">

**wb-core** — maintained by [nulswa](https://github.com/nulswa)

</div>
