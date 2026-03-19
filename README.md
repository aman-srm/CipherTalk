# 🔐 CipherTalk — End-to-End Encrypted Messaging

![CipherTalk Banner](https://img.shields.io/badge/CipherTalk-E2E%20Encrypted-00d4ff?style=for-the-badge&logo=lock&logoColor=white)
![Status](https://img.shields.io/badge/Status-Live-00ff88?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-a855f7?style=for-the-badge)
![Built With](https://img.shields.io/badge/Built%20With-Vanilla%20JS-ffb800?style=for-the-badge)

> A zero-knowledge, browser-native secure messaging application with military-grade encryption. No servers ever see your messages — everything is encrypted and decrypted exclusively in your browser.

**[🚀 Live Demo](https://your-vercel-url.vercel.app)** &nbsp;|&nbsp; **[📖 Documentation](./docs/TECHNICAL.md)** &nbsp;|&nbsp; **[🔒 Security](./docs/SECURITY.md)**

---

## ✨ Features

### Security & Encryption
- **ECDH P-256** asymmetric key exchange — a unique shared key per conversation
- **AES-GCM-256** symmetric encryption for every message and file
- **PBKDF2** with 200,000 iterations for passphrase-derived key protection
- **Zero-knowledge architecture** — Supabase servers store only ciphertext, never plaintext
- **Public key fingerprint verification** — SHA-256 fingerprints to verify contact identity out-of-band
- **Tab lock** — session locks automatically when you switch browser tabs

### Messaging
- Real-time message delivery via Supabase WebSocket (with polling fallback)
- **Temporary decrypt sessions** — enter passphrase once, all messages auto-decrypt for 30 seconds, then re-encrypt
- Decrypt your own sent messages using your passphrase
- Message retention modes: **View Once**, **24 Hours**, **Keep**
- View-once content-aware timer (adjusts based on word count, media type)
- Server-side automatic deletion via pg_cron scheduled jobs
- Clear chat functionality

### Friends & Discovery
- Username-based user search with public key fingerprint display
- Friend request system (send, accept, decline)
- Real-time request badge notifications

### Media & Files
- Send encrypted photos, videos, audio, and documents (up to 25MB)
- All files encrypted with AES-GCM before leaving the browser
- Encrypted storage via Supabase Storage
- Image lightbox viewer
- Audio and video playback after decryption

### Cross-Device & Mobile
- **Cross-device login** — encrypted private key backed up to Supabase, accessible from any device
- Fully responsive — mobile-first design with slide-in sidebar navigation
- Progressive Web App ready — add to home screen on iOS/Android
- iOS zoom prevention and native-feel touch targets

### UI & Experience
- Cyberpunk-inspired dark interface with animated Matrix rain background
- Interactive canvas with hex cells, circuit traces, particle network, mouse-reactive animations
- Send burst and decrypt ripple visual effects
- Smooth animated transitions throughout

---

## 🏗️ Architecture

```
Browser (Client)                    Supabase (Server)
─────────────────                   ─────────────────
Key Generation (ECDH P-256)         auth.users (email + JWT)
Passphrase → PBKDF2 → AES key       profiles (username, public_key,
Private key encrypted locally                encrypted_private_key)
                                    friendships (status: pending/accepted)
Message typed by user               messages (ciphertext only — no plaintext)
    ↓ AES-GCM-256 encrypt           secure-files (encrypted blobs)
    ↓ ciphertext sent to Supabase   
    ↓ Supabase stores ciphertext    pg_cron (auto-deletes expired messages)
    ↓ Recipient receives ciphertext
    ↓ AES-GCM-256 decrypt in browser
Message displayed as plaintext
```

**Key security property:** The server stores only ciphertext. Even if Supabase is fully compromised, no messages can be read without the user's passphrase.

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Vanilla HTML, CSS, JavaScript (ES Modules) |
| Encryption | Web Crypto API (browser-native, no libraries) |
| Backend | Supabase (PostgreSQL + Auth + Realtime + Storage) |
| Hosting | Vercel (static deployment) |
| Real-time | Supabase Realtime WebSocket + polling fallback |
| Scheduled Jobs | pg_cron (server-side message deletion) |

---

## 🚀 Getting Started

### Prerequisites
- A [Supabase](https://supabase.com) account (free tier)
- A [Vercel](https://vercel.com) account (free tier)
- A GitHub account

### 1. Set Up Supabase

Create a new Supabase project, then run the following SQL in the SQL Editor:

```sql
-- Profiles table
create table public.profiles (
  id uuid primary key references auth.users(id),
  username text unique not null,
  display_name text not null,
  public_key text not null,
  encrypted_private_key text,
  created_at timestamptz default now()
);

-- Friendships table
create table public.friendships (
  id uuid primary key default gen_random_uuid(),
  requester_id uuid references public.profiles(id),
  addressee_id uuid references public.profiles(id),
  status text default 'pending',
  created_at timestamptz default now(),
  updated_at timestamptz default now(),
  unique(requester_id, addressee_id)
);

-- Messages table
create table public.messages (
  id uuid primary key default gen_random_uuid(),
  sender_id uuid references public.profiles(id),
  recipient_id uuid references public.profiles(id),
  ciphertext text not null,
  message_type text default 'text',
  file_name text,
  file_size bigint,
  file_mime text,
  storage_path text,
  retention text default 'keep',
  viewed_at timestamptz,
  expires_at timestamptz,
  deleted_by_sender boolean default false,
  deleted_by_recipient boolean default false,
  created_at timestamptz default now()
);

-- Enable Row Level Security
alter table public.profiles enable row level security;
alter table public.friendships enable row level security;
alter table public.messages enable row level security;

-- Profiles policies
create policy "Public profiles readable" on public.profiles for select using (true);
create policy "Users insert own profile" on public.profiles for insert with check (auth.uid() = id);
create policy "Users update own profile" on public.profiles for update using (auth.uid() = id);

-- Friendships policies
create policy "Users read own friendships" on public.friendships for select using (auth.uid() = requester_id or auth.uid() = addressee_id);
create policy "Users insert friendships" on public.friendships for insert with check (auth.uid() = requester_id);
create policy "Users update own friendships" on public.friendships for update using (auth.uid() = requester_id or auth.uid() = addressee_id);
create policy "Users delete own friendships" on public.friendships for delete using (auth.uid() = requester_id or auth.uid() = addressee_id);

-- Messages policies
create policy "Users read own messages" on public.messages for select using (auth.uid() = sender_id or auth.uid() = recipient_id);
create policy "Users insert messages" on public.messages for insert with check (auth.uid() = sender_id);
create policy "Users update own messages" on public.messages for update using (auth.uid() = sender_id or auth.uid() = recipient_id);

-- Enable Realtime
alter publication supabase_realtime add table public.messages;

-- Username format constraint
alter table public.profiles add constraint username_format check (username ~ '^[a-z0-9_]{3,20}$');

-- Auto-deletion cron job
create extension if not exists pg_cron;
select cron.schedule(
  'delete-expired-messages',
  '* * * * *',
  $$
  delete from public.messages where
    (retention = 'once' and viewed_at is not null) or
    (retention = '24h' and expires_at < now()) or
    (deleted_by_sender = true and deleted_by_recipient = true and created_at < now() - interval '24 hours');
  $$
);
```

### 2. Create Storage Bucket

In Supabase → Storage → New bucket:
- Name: `secure-files`
- Public: No (private)

Then add storage policies:
```sql
create policy "auth_upload" on storage.objects for insert to authenticated with check (bucket_id = 'secure-files');
create policy "auth_download" on storage.objects for select to authenticated using (bucket_id = 'secure-files');
create policy "auth_delete" on storage.objects for delete to authenticated using (bucket_id = 'secure-files' and (storage.foldername(name))[1] = auth.uid()::text);
```

### 3. Configure the App

Open `index.html` and set your Supabase credentials:

```js
const SUPABASE_URL = 'https://YOUR_PROJECT_ID.supabase.co'
const SUPABASE_ANON_KEY = 'your_anon_key_here'
```

Find these values in Supabase → Settings → API.

### 4. Deploy to Vercel

```bash
# Push to GitHub
git init
git add index.html
git commit -m "Initial deployment"
git remote add origin https://github.com/YOUR_USERNAME/CipherTalk.git
git push -u origin main
```

Then connect your GitHub repo to Vercel — it deploys automatically.

Finally, add your Vercel URL to Supabase → Authentication → URL Configuration → Site URL.

---

## 📁 Repository Structure

```
CipherTalk/
├── index.html          # Complete application (single-file architecture)
├── README.md           # This file
├── LICENSE             # MIT License
└── docs/
    ├── TECHNICAL.md    # Cryptographic architecture deep-dive
    ├── SECURITY.md     # Security model and threat analysis
    └── SETUP.md        # Detailed setup guide with screenshots
```

---

## 🔒 Security Model

CipherTalk follows a **zero-knowledge** design:

| What the server knows | What the server does NOT know |
|----------------------|-------------------------------|
| Your email address | Your passphrase |
| Your username | Your private key |
| Your public key | Message content |
| That a message exists | File contents |
| Message timestamps | Who your friends are (by content) |

The encrypted private key stored in Supabase is protected by your passphrase via PBKDF2 (200,000 iterations). Without your passphrase, it is computationally infeasible to recover.

See [SECURITY.md](./docs/SECURITY.md) for full threat model analysis.

---

## 📸 Screenshots

> *Add screenshots of your app here after deployment*
> 
> Suggested: Login screen, chat interface, decrypt modal, mobile view

---

## 🗺️ Roadmap

- [ ] Rate limiting on account creation
- [ ] Google Sign-In (OAuth)
- [ ] Push notifications
- [ ] Message search
- [ ] Read receipts
- [ ] Profile pictures
- [ ] Group chats

---

## 📄 License

MIT License — see [LICENSE](./LICENSE) for details.

---

## 👤 Author

**Aman** — [@aman-srm](https://github.com/aman-srm)

*Built entirely with vanilla JavaScript and the browser-native Web Crypto API — no encryption libraries.*
