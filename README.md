# omp-uptime

Monitor eksternal untuk OMP VPS (korea-vps, Azure Korea Central).

## Kenapa repo ini ada

15 Nov 2026 VPS beku ~10 jam (12:36 → 22:45 UTC) karena sisi host Azure:
`%iowait 70%`, `%steal 28%`, kanal KVP host↔guest putus, dan RTC host melompat
maju → jam guest lompat → 63 slot cron catch-up tembak bareng (load 479).

Masalahnya: **box yang beku tidak bisa melaporkan dirinya sendiri.** Watchdog
di dalam box baru sadar setelah semuanya thaw — 10 jam kemudian.

Jadi pemantaunya harus hidup di luar box. Repo ini berisi:

1. `/.github/workflows/uptime.yml` — GitHub Actions tiap 10 menit, menguji
   endpoint publik VPS +-age heartbeat commit-status dari VPS.
2. Issue `VPS DOWN: korea-vps (OMP)` dibuat otomatis saat tidak sehat,
   ditutup otomatis saat pulih, plus notifikasi Telegram.

## Sinyal yang dipantau

| Sinyal | Arti |
| --- | --- |
| HTTP 200/3xx/401/403 di `prl.`, `ffbot.`, `9router.rioganteng.my.id` | tunnel + service di dalam box hidup |
| commit-status `omp-vps-heartbeat` di repo ini, umur < 15 menit | **box sendiri** masih bisa jalan cron dan Internet |

Sinyal kedua yang pivotal: kalau box beku total, `curl` dari GitHub ke
`20.194.27.169` timeout **dan** heartbeat commit-status jadi basi. Dua-duanya
menunjuk beku, bukan sekadar service yang mati.

## Cara heartbeat dikirim

Script di VPS:

```
~/.omp/agent/ops/cron/heartbeat.sh
```

Dijadwalkan `omp-cron-heartbeat.timer` tiap 5 menit. Scriptnya POST commit
status ke SHA tetap di repo ini (bukan ke branch), jadi tidak pernah konflik
dengan push workflow atau repo lain.

## Rahasia

`TELEGRAM_BOT_TOKEN` dan `TELEGRAM_CHAT_ID` disimpan sebagai Actions secret
repo ini. Tidak pernah ditulis ke file yang di-commit.

## Yang TIDAK dicakup

Workflow ini hanya deteksi. Root cause 15 Nov ada di sisi host Azure
(fabric/PTP/storage) — tidak bisa difix dari dalam guest. Yang bisa dipatch
dari sini: NTP fallback (`/etc/chrony/sources.d/omp-fallback.sources`),
`flock` di cron yang bisa numpuk, dan sebaran timer systemd.
