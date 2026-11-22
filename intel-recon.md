# MUSE BUG BOUNTY — INTEL REKON (16 Nov 2026)

## Target & Scope
- Web: muse.ai (Next.js di Vercel, server: Vercel, x-vercel-id: sin1)
- Auth: auth.meta.com (OIDC), form-action mencakup https://*.od.auth.meta.com
- App: iOS/Android (Android package: com.facebook.aura)
- Program: bugbounty.meta.com, kategori "Muse", max $300k
- Catatan: ~630 submission, 64 valid, $1.2M terbayar per 2 Nov 2026 → SNR rendah, report harus BUKTI end-to-end

## Arsitektur (dari blog resmi — "Hatch" = codename Muse)
- 1 VM per user (Linux, systemd-nspawn "runtime cell"); root di cell = user tidak-privileged di host
- Runtime cell: filesystem Debian terpisah, vNIC sendiri, syscall difilter (io_uring DIBLOKIR),
  caps dibatasi (no CAP_SYS_PTRACE, no CAP_NET_ADMIN)
- Layanan host-side (LUAR cell, systemd units terpisah):
  * hatch-safety  : classifier prompt injection (ensemble), jalan di luar cell
  * privsep       : worker konektor, credential allowlist per-cgroup
  * hatch-authd   : simpan OAuth token/credentials di VM, "credential surrogation"
  * Sentinel      : OTORITAS TUNGGAL izin konektor + egress jaringan; JIT credential insertion
- IPC: Unix domain sockets + SO_PEERCRED + peer ACL
- Egress: SSRF guard (DNS rebinding dilindungi), eBPF cgroup interception,
  "tainted egress" = data-flow tracking kernel-level (eBPF LSM hooks)
- Browser: CDP dibroker di luar cell; sub-agent hanya lihat accessibility tree, TIDAK bisa JS,
  devtools disabled; agent di-PAUSE saat user takeover / autofill kredensial
- Email connector: filter OTP/password-reset/magic-link (deterministik + classifier)

## Histori bug yang terbayar (pola!)
- $300k: trusted header dari local HTTPS proxy dipercaya sebagai auth; layer proxy kedua muncul
  saat migrasi → header bisa dipalsukan dari luar → baca VM pre-release. Pelajaran: JANGAN percaya
  header/loopback; cek DOUBLE-PROXY / header spoofing di semua jalur internal
- $90k: prompt injection meniru STATUS MESSAGES tool browsing milik Muse sendiri → agent eksekusi
  aksi terlarang. Pelajaran: imitasi UI status tool = vektor kuat
- $90k: bypass redaksi OTP via formatting tertentu di email
- $50k: Android app — link handler percaya shared-page path tanpa validasi host → web view
  privileged bocorkan kredensial
- $1k: Mac app — auto-update abuse → privesc lokal

## Attack surface prioritas (rencana riset)
1. [C6 $7.5k] Cell → host escape: sistemd-nspawn, syscall filter, caps, eBPF taint
   - fokus: SO_PEERCRED unix sockets, surrogate token handling, egress path
2. [C7 $5k] Sentinel bypass: self-approval, connector permission "ask"→"allow",
   egress laundering via allowlisted destination / poppy.json business endpoints / scheduled jobs
3. [C3 $130k] Prompt injection → full chain: konten ditanam di mana agent membaca → kirim
   memori/email user lain ATAU self-egress. Catatan: injection murni tidak dibayar, HARUS chain
4. [C5 $10k] Fleet secrets: model provider API key di VM; poppy.json/MCP integrasi
5. Web app muse.ai: auth.meta.com OIDC flow, CSP ketat (nonce+strict-dynamic), Stripe checkout

## Endpoint menarik
- wss://gateway.meta.ai, wss://edge-chat(-latest).meta.ai, wss://shortwave.meta.ai
- *.metaaiusercontent.com, *.ecto1usercontent.com (artifact hosting — cek XSS/fetch)
- *.meta-agents-apps.workers.dev (Cloudflare workers — frame-src)
- personalagentprotocol.org (Poppy v1.0 — poppy.json spec)

## Out of scope (JANGAN buang waktu)
- jailbreak/system prompt extraction/DoS/rate-limit bypass/hallucination
- CPU/kernel CVE publik < 30 hari sejak disclosure
- egress yang "permitted by design"
