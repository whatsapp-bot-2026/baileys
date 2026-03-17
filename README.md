# @whatsappbot/baileys — Patched Edition

> Fork dari `@whatsappbot/baileys` dengan patch penuh untuk **Interactive Message** (semua tipe button), support shorthand API yang lebih bersih, dan berbagai contoh siap pakai.

---

## Daftar Isi

- [Instalasi](#instalasi)
- [Setup Koneksi](#setup-koneksi)
  - [QR Code](#qr-code)
  - [Pairing Code](#pairing-code)
  - [Custom Pairing Code](#custom-pairing-code)
- [Pesan Teks](#pesan-teks)
- [Pesan Media](#pesan-media)
  - [Gambar](#gambar)
  - [Video](#video)
  - [Audio / PTT](#audio--ptt)
  - [Dokumen](#dokumen)
  - [Stiker](#stiker)
- [Button Interaktif](#button-interaktif)
  - [Button URL (cta\_url)](#button-url-cta_url)
  - [Button Copy (cta\_copy)](#button-copy-cta_copy)
  - [Button Reply (quick\_reply)](#button-reply-quick_reply)
  - [Button Telepon (cta\_call)](#button-telepon-cta_call)
  - [Multi Button](#multi-button)
  - [Button + Gambar](#button--gambar)
  - [Button + Gambar dari URL](#button--gambar-dari-url)
  - [Button + Gambar dari File Lokal](#button--gambar-dari-file-lokal)
  - [Button + Video](#button--video)
- [List Message](#list-message)
  - [List Sederhana](#list-sederhana)
  - [List Multi Section](#list-multi-section)
- [Reply / Quote Pesan](#reply--quote-pesan)
- [Mention](#mention)
- [React Emoji](#react-emoji)
- [Delete Pesan](#delete-pesan)
- [Edit Pesan](#edit-pesan)
- [Poll](#poll)
- [Kontak](#kontak)
- [Lokasi](#lokasi)
- [Group](#group)
  - [Buat Group](#buat-group)
  - [Tambah Peserta](#tambah-peserta)
  - [Kick Peserta](#kick-peserta)
  - [Promote / Demote Admin](#promote--demote-admin)
  - [Update Info Group](#update-info-group)
  - [Link Invite Group](#link-invite-group)
- [Newsletter / Channel](#newsletter--channel)
  - [Cek Info Channel](#cek-info-channel)
  - [Subscribe Channel](#subscribe-channel)
- [Presence (Online / Mengetik)](#presence-online--mengetik)
- [Read Pesan](#read-pesan)
- [Download Media](#download-media)
- [Store (In-Memory)](#store-in-memory)
- [JID Utilities](#jid-utilities)
- [Events Lengkap](#events-lengkap)

---

## Instalasi

```bash
npm install @whatsappbot/baileys
```

atau clone + pasang manual:

```bash
cp -r package/ node_modules/@whatsappbot/baileys/
```

---

## Setup Koneksi

### QR Code

```js
import makeWASocket, {
  useMultiFileAuthState,
  fetchLatestBaileysVersion,
  makeCacheableSignalKeyStore,
  DisconnectReason
} from '@whatsappbot/baileys'
import { Boom } from '@hapi/boom'
import pino from 'pino'
import qrcode from 'qrcode-terminal'

const logger = pino({ level: 'silent' })

const startBot = async () => {
  const { state, saveCreds } = await useMultiFileAuthState('./session')
  const { version } = await fetchLatestBaileysVersion()

  const sock = makeWASocket({
    version,
    logger,
    printQRInTerminal: false,
    auth: {
      creds: state.creds,
      keys: makeCacheableSignalKeyStore(state.keys, logger)
    }
  })

  sock.ev.on('connection.update', ({ qr, connection, lastDisconnect }) => {
    if (qr) {
      console.clear()
      qrcode.generate(qr, { small: true })
    }

    if (connection === 'close') {
      const code = lastDisconnect?.error instanceof Boom
        ? lastDisconnect.error.output.statusCode
        : 0
      if (code !== DisconnectReason.loggedOut) startBot()
    }

    if (connection === 'open') {
      console.log('✅ Terhubung!')
    }
  })

  sock.ev.on('creds.update', saveCreds)
}

startBot()
```

---

### Pairing Code

```js
const sock = makeWASocket({
  version,
  logger,
  printQRInTerminal: false,
  auth: {
    creds: state.creds,
    keys: makeCacheableSignalKeyStore(state.keys, logger)
  }
})

if (!sock.authState.creds.registered) {
  setTimeout(async () => {
    const code = await sock.requestPairingCode('628xxxxxxxxxx')
    console.log('🔑 Pairing Code:', code)
  }, 3000)
}
```

---

### Custom Pairing Code

Pairing code dengan input dari terminal:

```js
import readline from 'readline'

const ask = (prompt) =>
  new Promise(resolve => {
    const rl = readline.createInterface({ input: process.stdin, output: process.stdout })
    rl.question(prompt, ans => { rl.close(); resolve(ans.trim()) })
  })

const startWithPairing = async () => {
  const number = await ask('Masukkan nomor (628xxx): ')
  const clean  = number.replace(/\D/g, '')

  const sock = makeWASocket({
    version,
    logger,
    printQRInTerminal: false,
    auth: {
      creds: state.creds,
      keys: makeCacheableSignalKeyStore(state.keys, logger)
    }
  })

  if (!sock.authState.creds.registered) {
    setTimeout(async () => {
      const code = await sock.requestPairingCode(clean)
      console.log('\n🔑 Pairing Code :', code)
      console.log('⏳ Masukkan kode ini di WhatsApp kamu\n')
    }, 3000)
  }

  sock.ev.on('creds.update', saveCreds)
  sock.ev.on('connection.update', ({ connection }) => {
    if (connection === 'open') console.log('✅ Berhasil terhubung dengan pairing code!')
  })
}
```

---

## Pesan Teks

```js
await sock.sendMessage('628xxx@s.whatsapp.net', {
  text: 'Halo dari bot!'
})
```

**Dengan format bold, italic, strikethrough:**

```js
await sock.sendMessage(jid, {
  text: '*Bold* _italic_ ~strikethrough~ `monospace`'
})
```

**Dengan link preview:**

```js
await sock.sendMessage(jid, {
  text: 'Cek ini: https://github.com',
  linkPreview: undefined
})
```

---

## Pesan Media

### Gambar

```js
await sock.sendMessage(jid, {
  image: { url: 'https://example.com/foto.jpg' },
  caption: 'Ini caption gambar'
})
```

**Dari buffer lokal:**

```js
import fs from 'fs'

await sock.sendMessage(jid, {
  image: fs.readFileSync('./gambar.jpg'),
  caption: 'Gambar dari file lokal'
})
```

---

### Video

```js
await sock.sendMessage(jid, {
  video: { url: 'https://example.com/video.mp4' },
  caption: 'Ini caption video',
  gifPlayback: false
})
```

**Sebagai GIF:**

```js
await sock.sendMessage(jid, {
  video: fs.readFileSync('./animasi.mp4'),
  gifPlayback: true,
  caption: 'Ini GIF'
})
```

---

### Audio / PTT

```js
await sock.sendMessage(jid, {
  audio: { url: 'https://example.com/audio.mp3' },
  mimetype: 'audio/mp4',
  ptt: false
})
```

**Voice note (PTT):**

```js
await sock.sendMessage(jid, {
  audio: fs.readFileSync('./suara.ogg'),
  mimetype: 'audio/ogg; codecs=opus',
  ptt: true
})
```

---

### Dokumen

```js
await sock.sendMessage(jid, {
  document: fs.readFileSync('./file.pdf'),
  mimetype: 'application/pdf',
  fileName: 'dokumen.pdf',
  caption: 'Ini file PDF'
})
```

---

### Stiker

```js
await sock.sendMessage(jid, {
  sticker: fs.readFileSync('./stiker.webp')
})
```

---

## Button Interaktif

> Semua tipe button menggunakan format shorthand yang sudah di-patch oleh package ini.

### Button URL (cta_url)

Tombol yang membuka link di browser.

```js
await sock.sendMessage(jid, {
  interactiveMessage: {
    title: '🌐 Buka Website',
    body: 'Klik tombol untuk membuka link.',
    footer: 'Powered by Bot',
    buttons: [
      {
        name: 'cta_url',
        buttonParamsJson: JSON.stringify({
          display_text: 'Buka Google',
          url: 'https://google.com',
          merchant_url: 'https://google.com'
        })
      }
    ]
  }
})
```

---

### Button Copy (cta_copy)

Tombol yang meng-copy teks ke clipboard.

```js
await sock.sendMessage(jid, {
  interactiveMessage: {
    title: '📋 Copy Kode',
    body: 'Klik tombol untuk menyalin kode referral.',
    buttons: [
      {
        name: 'cta_copy',
        buttonParamsJson: JSON.stringify({
          display_text: 'Copy Kode',
          copy_code: 'REFERRAL-XYZ123'
        })
      }
    ]
  }
})
```

---

### Button Reply (quick_reply)

Tombol yang mengirim pesan balasan otomatis.

```js
await sock.sendMessage(jid, {
  interactiveMessage: {
    title: '❓ Konfirmasi',
    body: 'Apakah kamu yakin?',
    buttons: [
      {
        name: 'quick_reply',
        buttonParamsJson: JSON.stringify({
          display_text: '✅ Ya',
          id: 'confirm_yes'
        })
      },
      {
        name: 'quick_reply',
        buttonParamsJson: JSON.stringify({
          display_text: '❌ Tidak',
          id: 'confirm_no'
        })
      }
    ]
  }
})
```

---

### Button Telepon (cta_call)

Tombol yang langsung melakukan panggilan.

```js
await sock.sendMessage(jid, {
  interactiveMessage: {
    title: '📞 Hubungi Kami',
    body: 'Klik untuk menghubungi langsung.',
    buttons: [
      {
        name: 'cta_call',
        buttonParamsJson: JSON.stringify({
          display_text: 'Telepon Sekarang',
          phone_number: '+628xxxxxxxxxx'
        })
      }
    ]
  }
})
```

---

### Multi Button

Gabungkan berbagai tipe button dalam satu pesan.

```js
await sock.sendMessage(jid, {
  interactiveMessage: {
    title: '🎯 Menu Pilihan',
    body: 'Pilih salah satu aksi di bawah:',
    footer: 'Bot v1.0',
    buttons: [
      {
        name: 'cta_url',
        buttonParamsJson: JSON.stringify({
          display_text: '🌐 Buka Website',
          url: 'https://example.com'
        })
      },
      {
        name: 'cta_copy',
        buttonParamsJson: JSON.stringify({
          display_text: '📋 Copy Link',
          copy_code: 'https://example.com'
        })
      },
      {
        name: 'quick_reply',
        buttonParamsJson: JSON.stringify({
          display_text: '💬 Chat Admin',
          id: 'chat_admin'
        })
      }
    ]
  }
})
```

---

### Button + Gambar

Menampilkan gambar di header pesan button.

```js
await sock.sendMessage(jid, {
  interactiveMessage: {
    title: '🖼️ Gambar dengan Button',
    body: 'Header berupa gambar dari URL.',
    footer: 'Bot v1.0',
    image: {
      url: 'https://example.com/banner.jpg'
    },
    buttons: [
      {
        name: 'cta_url',
        buttonParamsJson: JSON.stringify({
          display_text: 'Lihat Selengkapnya',
          url: 'https://example.com'
        })
      }
    ]
  }
})
```

---

### Button + Gambar dari URL

```js
await sock.sendMessage(jid, {
  interactiveMessage: {
    title: '📸 Promo Hari Ini',
    body: 'Dapatkan diskon 50% untuk semua produk!',
    footer: 'Berlaku hari ini saja',
    image: {
      url: 'https://i.ibb.co/placeholder/promo.jpg'
    },
    buttons: [
      {
        name: 'cta_url',
        buttonParamsJson: JSON.stringify({
          display_text: '🛒 Beli Sekarang',
          url: 'https://tokopedia.com'
        })
      },
      {
        name: 'cta_copy',
        buttonParamsJson: JSON.stringify({
          display_text: '📋 Copy Kode Promo',
          copy_code: 'DISKON50'
        })
      }
    ]
  }
})
```

---

### Button + Gambar dari File Lokal

```js
import fs from 'fs'

const imgBuffer = fs.readFileSync('./assets/banner.jpg')

await sock.sendMessage(jid, {
  interactiveMessage: {
    title: '📦 Produk Baru',
    body: 'Gambar diambil dari file lokal.',
    image: {
      jpegThumbnail: imgBuffer
    },
    buttons: [
      {
        name: 'quick_reply',
        buttonParamsJson: JSON.stringify({
          display_text: '🛍️ Pesan Sekarang',
          id: 'order_now'
        })
      }
    ]
  }
})
```

---

### Button + Video

```js
await sock.sendMessage(jid, {
  interactiveMessage: {
    title: '🎬 Video Demo',
    body: 'Tonton video produk kami.',
    video: {
      url: 'https://example.com/demo.mp4'
    },
    buttons: [
      {
        name: 'cta_url',
        buttonParamsJson: JSON.stringify({
          display_text: '▶️ Tonton di YouTube',
          url: 'https://youtube.com'
        })
      }
    ]
  }
})
```

---

## List Message

### List Sederhana

```js
await sock.sendMessage(jid, {
  listMessage: {
    title: '📋 Menu Bot',
    description: 'Pilih salah satu opsi.',
    footerText: 'Bot v1.0',
    buttonText: '📂 Lihat Menu',
    listType: 1,
    sections: [
      {
        title: '🛠️ Umum',
        rows: [
          { title: 'Ping',  description: 'Cek kecepatan bot', rowId: '.ping' },
          { title: 'Info',  description: 'Info bot',           rowId: '.info' },
          { title: 'Help',  description: 'Bantuan',            rowId: '.help' }
        ]
      }
    ]
  }
})
```

---

### List Multi Section

```js
await sock.sendMessage(jid, {
  listMessage: {
    title: '📋 Menu Lengkap',
    description: 'Halo! Pilih menu yang kamu butuhkan.',
    footerText: 'Powered by Bot',
    buttonText: '☰ Buka Menu',
    listType: 1,
    sections: [
      {
        title: '🔧 Tools',
        rows: [
          { title: 'ToURL',    description: 'Upload media ke URL',    rowId: '.tourl' },
          { title: 'Sticker',  description: 'Buat stiker dari gambar', rowId: '.sticker' },
          { title: 'QR Code',  description: 'Generate QR Code',        rowId: '.qr' }
        ]
      },
      {
        title: '📡 Info',
        rows: [
          { title: 'Cuaca',    description: 'Cek cuaca hari ini',       rowId: '.cuaca' },
          { title: 'Kurs',     description: 'Kurs mata uang',           rowId: '.kurs' }
        ]
      },
      {
        title: '👑 Owner',
        rows: [
          { title: 'Broadcast', description: 'Kirim pesan massal',      rowId: '.bc' },
          { title: 'Restart',   description: 'Restart bot',             rowId: '.restart' }
        ]
      }
    ]
  }
})
```

---

## Reply / Quote Pesan

```js
sock.ev.on('messages.upsert', async ({ messages }) => {
  const msg = messages[0]

  await sock.sendMessage(
    msg.key.remoteJid,
    { text: 'Ini balasan untuk pesanmu!' },
    { quoted: msg }
  )
})
```

---

## Mention

```js
const jid = '628xxx@s.whatsapp.net'

await sock.sendMessage(groupJid, {
  text: `Halo @${jid.split('@')[0]}!`,
  mentions: [jid]
})
```

**Mention semua (tag all):**

```js
const meta     = await sock.groupMetadata(groupJid)
const all      = meta.participants.map(p => p.id)
const tagText  = all.map(p => `@${p.split('@')[0]}`).join(' ')

await sock.sendMessage(groupJid, {
  text: `📢 Perhatian semua!\n${tagText}`,
  mentions: all
})
```

---

## React Emoji

```js
await sock.sendMessage(jid, {
  react: {
    text: '❤️',
    key: msg.key
  }
})
```

**Hapus react:**

```js
await sock.sendMessage(jid, {
  react: {
    text: '',
    key: msg.key
  }
})
```

---

## Delete Pesan

**Hapus pesan sendiri:**

```js
await sock.sendMessage(jid, {
  delete: msg.key
})
```

---

## Edit Pesan

```js
await sock.sendMessage(jid, {
  text: 'Teks yang sudah diedit',
  edit: msg.key
})
```

---

## Poll

```js
await sock.sendMessage(jid, {
  poll: {
    name: 'Pilih warna favoritmu?',
    values: ['🔴 Merah', '🔵 Biru', '🟢 Hijau', '🟡 Kuning'],
    selectableCount: 1
  }
})
```

**Poll multi pilih:**

```js
await sock.sendMessage(jid, {
  poll: {
    name: 'Fitur apa yang kamu mau?',
    values: ['Button', 'Stiker', 'AI Chat', 'Music DL'],
    selectableCount: 2
  }
})
```

---

## Kontak

```js
await sock.sendMessage(jid, {
  contacts: {
    displayName: 'John Doe',
    contacts: [
      {
        displayName: 'John Doe',
        vcard: `BEGIN:VCARD\nVERSION:3.0\nFN:John Doe\nTEL;type=CELL;type=VOICE;waid=628xxx:+628xxx\nEND:VCARD`
      }
    ]
  }
})
```

---

## Lokasi

```js
await sock.sendMessage(jid, {
  location: {
    degreesLatitude: -6.2088,
    degreesLongitude: 106.8456,
    name: 'Monas, Jakarta',
    address: 'Jl. Medan Merdeka Barat, Jakarta Pusat'
  }
})
```

---

## Group

### Buat Group

```js
const group = await sock.groupCreate(
  'Nama Group',
  ['628aaa@s.whatsapp.net', '628bbb@s.whatsapp.net']
)
console.log('Group JID:', group.id)
```

---

### Tambah Peserta

```js
await sock.groupParticipantsUpdate(
  'groupId@g.us',
  ['628xxx@s.whatsapp.net'],
  'add'
)
```

---

### Kick Peserta

```js
await sock.groupParticipantsUpdate(
  'groupId@g.us',
  ['628xxx@s.whatsapp.net'],
  'remove'
)
```

---

### Promote / Demote Admin

```js
await sock.groupParticipantsUpdate(
  'groupId@g.us',
  ['628xxx@s.whatsapp.net'],
  'promote'
)

await sock.groupParticipantsUpdate(
  'groupId@g.us',
  ['628xxx@s.whatsapp.net'],
  'demote'
)
```

---

### Update Info Group

```js
await sock.groupUpdateSubject('groupId@g.us', 'Nama Group Baru')

await sock.groupUpdateDescription('groupId@g.us', 'Deskripsi group baru.')

await sock.groupSettingUpdate('groupId@g.us', 'announcement')
await sock.groupSettingUpdate('groupId@g.us', 'not_announcement')
await sock.groupSettingUpdate('groupId@g.us', 'locked')
await sock.groupSettingUpdate('groupId@g.us', 'unlocked')
```

---

### Link Invite Group

```js
const code = await sock.groupInviteCode('groupId@g.us')
console.log('Link:', `https://chat.whatsapp.com/${code}`)

await sock.groupRevokeInvite('groupId@g.us')
```

**Ambil metadata group:**

```js
const meta = await sock.groupMetadata('groupId@g.us')
console.log(meta.subject)
console.log(meta.participants)
```

---

## Newsletter / Channel

### Cek Info Channel

```js
const info = await sock.newsletterMetadata('invite', 'INVITE_CODE_DARI_LINK')
console.log('Nama    :', info.name)
console.log('ID      :', info.id)
console.log('Subs    :', info.subscriberCount)
console.log('Verified:', info.verification)
```

**Dari link channel:**

```js
const link       = 'https://whatsapp.com/channel/0029VaXXXX'
const inviteCode = link.match(/channel\/([a-zA-Z0-9_-]+)/)?.[1]
const info       = await sock.newsletterMetadata('invite', inviteCode)
```

---

### Subscribe Channel

```js
await sock.newsletterFollow(channelJid)

await sock.newsletterUnfollow(channelJid)
```

---

## Presence (Online / Mengetik)

```js
await sock.sendPresenceUpdate('available',  jid)
await sock.sendPresenceUpdate('unavailable', jid)
await sock.sendPresenceUpdate('composing',   jid)
await sock.sendPresenceUpdate('recording',   jid)
await sock.sendPresenceUpdate('paused',      jid)
```

**Subscribe ke presence orang lain:**

```js
await sock.presenceSubscribe(jid)

sock.ev.on('presence.update', ({ id, presences }) => {
  console.log(id, presences)
})
```

---

## Read Pesan

```js
await sock.readMessages([msg.key])
```

**Auto-read semua pesan:**

```js
sock.ev.on('messages.upsert', async ({ messages }) => {
  for (const msg of messages) {
    if (!msg.key.fromMe) {
      await sock.readMessages([msg.key])
    }
  }
})
```

---

## Download Media

```js
import { downloadMediaMessage } from '@whatsappbot/baileys'

sock.ev.on('messages.upsert', async ({ messages }) => {
  const msg = messages[0]
  if (!msg.message) return

  const type = Object.keys(msg.message)[0]

  if (['imageMessage', 'videoMessage', 'stickerMessage'].includes(type)) {
    const buffer = await downloadMediaMessage(msg, 'buffer', {})
    require('fs').writeFileSync(`./downloads/${Date.now()}.jpg`, buffer)
    console.log('✅ Media berhasil diunduh')
  }
})
```

**Unduh dari quoted message:**

```js
import { downloadMediaMessage, getContentType } from '@whatsappbot/baileys'

const quotedMsg  = msg.message?.extendedTextMessage?.contextInfo?.quotedMessage
const quotedType = getContentType(quotedMsg)

const buffer = await downloadMediaMessage(
  {
    key: { remoteJid: msg.key.remoteJid, id: msg.message?.extendedTextMessage?.contextInfo?.stanzaId, fromMe: false },
    message: quotedMsg
  },
  'buffer',
  {}
)
```

---

## Store (In-Memory)

```js
import { makeInMemoryStore } from '@whatsappbot/baileys'
import pino from 'pino'

const logger = pino({ level: 'silent' })
const store  = makeInMemoryStore({ logger })

store.bind(sock.ev)

const msg = await store.loadMessage(jid, messageId)
```

---

## JID Utilities

```js
import { jidNormalizedUser, jidDecode, areJidsSameUser, isJidGroup } from '@whatsappbot/baileys'

const jid = jidNormalizedUser('628xxx@s.whatsapp.net')

const decoded = jidDecode('628xxx@s.whatsapp.net')
console.log(decoded.user, decoded.server)

const same = areJidsSameUser('628xxx@s.whatsapp.net', '628xxx@lid')

const isGroup = isJidGroup('1234567890@g.us')
```

**Konversi nomor ke JID (dari patch main.js):**

```js
sock.toJid  = (number) => `${String(number).replace(/\D/g, '')}@s.whatsapp.net`
sock.toLid  = (number) => `${String(number).replace(/\D/g, '')}@lid`
sock.toPn   = (jid)    => jid.split('@')[0] + '@s.whatsapp.net'

const userJid  = sock.toJid('628123456789')
const userLid  = sock.toLid('628123456789')
```

---

## Events Lengkap

```js
sock.ev.on('connection.update',    (update) => { })

sock.ev.on('creds.update',         (creds)  => { })

sock.ev.on('messages.upsert',      ({ messages, type }) => { })

sock.ev.on('messages.update',      (updates) => { })

sock.ev.on('messages.delete',      ({ keys }) => { })

sock.ev.on('message-receipt.update', (updates) => { })

sock.ev.on('messages.reaction',    (reactions) => { })

sock.ev.on('presence.update',      ({ id, presences }) => { })

sock.ev.on('chats.upsert',         (chats)  => { })
sock.ev.on('chats.update',         (chats)  => { })
sock.ev.on('chats.delete',         (ids)    => { })

sock.ev.on('contacts.upsert',      (contacts) => { })
sock.ev.on('contacts.update',      (contacts) => { })

sock.ev.on('groups.upsert',        (groups)  => { })
sock.ev.on('groups.update',        (updates) => { })

sock.ev.on('group-participants.update', ({ id, participants, action }) => {
  console.log(`Action ${action} di group ${id}`)
  console.log('Peserta:', participants)
})

sock.ev.on('blocklist.set',    ({ blocklist }) => { })
sock.ev.on('blocklist.update', ({ blocklist, type }) => { })

sock.ev.on('call',             (calls) => { })

sock.ev.on('labels.edit',      (label)   => { })
sock.ev.on('labels.association', ({ association, type }) => { })
```

---

## Contoh Bot Minimal Lengkap

```js
import makeWASocket, {
  useMultiFileAuthState,
  fetchLatestBaileysVersion,
  makeCacheableSignalKeyStore,
  DisconnectReason,
  getContentType
} from '@whatsappbot/baileys'
import { Boom } from '@hapi/boom'
import pino from 'pino'

const logger = pino({ level: 'silent' })

const startBot = async () => {
  const { state, saveCreds } = await useMultiFileAuthState('./session')
  const { version } = await fetchLatestBaileysVersion()

  const sock = makeWASocket({
    version,
    logger,
    printQRInTerminal: true,
    auth: {
      creds: state.creds,
      keys: makeCacheableSignalKeyStore(state.keys, logger)
    }
  })

  sock.ev.on('creds.update', saveCreds)

  sock.ev.on('connection.update', ({ connection, lastDisconnect }) => {
    if (connection === 'close') {
      const code = lastDisconnect?.error instanceof Boom
        ? lastDisconnect.error.output.statusCode : 0
      if (code !== DisconnectReason.loggedOut) startBot()
    }
    if (connection === 'open') console.log('✅ Bot aktif!')
  })

  sock.ev.on('messages.upsert', async ({ messages, type }) => {
    if (type !== 'notify') return

    for (const msg of messages) {
      if (!msg.message || msg.key.fromMe) continue

      const jid      = msg.key.remoteJid
      const msgType  = getContentType(msg.message)
      const body     = msg.message?.conversation
        || msg.message[msgType]?.text
        || msg.message[msgType]?.caption
        || ''

      if (body === '.ping') {
        await sock.sendMessage(jid, { text: '🏓 Pong!' }, { quoted: msg })
      }

      if (body === '.menu') {
        await sock.sendMessage(jid, {
          interactiveMessage: {
            title: '📋 Menu',
            body: 'Pilih opsi:',
            footer: 'Bot v1.0',
            buttons: [
              {
                name: 'quick_reply',
                buttonParamsJson: JSON.stringify({ display_text: '🏓 Ping', id: '.ping' })
              },
              {
                name: 'cta_url',
                buttonParamsJson: JSON.stringify({ display_text: '🌐 GitHub', url: 'https://github.com' })
              }
            ]
          }
        }, { quoted: msg })
      }
    }
  })
}

startBot()
```

---

## Lisensi

MIT — [@whatsappbot/baileys](https://github.com/whatsapp-bot-2026/baileys)

> Dibuat oleh **Dayat** — 2026
