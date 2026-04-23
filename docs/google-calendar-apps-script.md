# Setup Google Calendar Webhook

Dashboard ini mengirim data izin ke Google Calendar melalui Web App Google Apps Script.
Request dikirim sebagai body teks berisi JSON agar tetap aman dipanggil langsung dari browser tanpa preflight CORS tambahan.

## Payload dari dashboard

Dashboard akan mengirim JSON seperti ini:

```json
{
  "feature_key": "feature-id:28",
  "permit_key": "12170007219530022",
  "permit_name": "CV. Bumi Harapan Sejahtera",
  "permit_type": "IUP OP",
  "permit_status": "Aktif",
  "nomor_sk": "12170007219530022",
  "masa_berlaku": "5 Tahun",
  "tanggal_sk": "2024-08-26",
  "expiry_date": "2029-08-26",
  "events": [
    {
      "event_key": "feature-id:28::reminder-6-months",
      "title": "Pengingat 6 Bulan Sebelum Izin Berakhir - CV. Bumi Harapan Sejahtera",
      "start_date": "2029-02-26",
      "end_date": "2029-02-26",
      "description": "..."
    }
  ]
}
```

## Apps Script contoh

Buka `script.google.com`, buat project baru, lalu isi `Code.gs` dengan script berikut:

```javascript
const CALENDAR_ID = 'primary';

function doPost(e) {
  const payload = JSON.parse(e.postData.contents || '{}');
  const calendar = CalendarApp.getCalendarById(CALENDAR_ID);

  if (!calendar) {
    return jsonResponse({ ok: false, message: 'Calendar tidak ditemukan.' });
  }

  const events = Array.isArray(payload.events) ? payload.events : [];

  events.forEach(item => {
    upsertAllDayEvent_(calendar, payload, item);
  });

  return jsonResponse({
    ok: true,
    message: 'Sinkronisasi berhasil.',
    total_events: events.length
  });
}

function upsertAllDayEvent_(calendar, payload, item) {
  const marker = '[DASHBOARD-MINERBA:' + item.event_key + ']';
  const startDate = new Date(item.start_date + 'T00:00:00');
  const endDate = new Date(item.end_date + 'T00:00:00');
  endDate.setDate(endDate.getDate() + 1);

  const existing = calendar
    .getEventsForDay(startDate)
    .find(event => (event.getDescription() || '').indexOf(marker) !== -1);

  const description = [
    marker,
    item.description || '',
    '',
    'Permit Key: ' + (payload.permit_key || '-'),
    'Feature Key: ' + (payload.feature_key || '-')
  ].join('\n');

  if (existing) {
    existing.setTitle(item.title || 'Pengingat Izin');
    existing.setDescription(description);
    if (item.location) {
      existing.setLocation(item.location);
    }
    return;
  }

  calendar.createAllDayEvent(item.title || 'Pengingat Izin', startDate, endDate, {
    description,
    location: item.location || ''
  });
}

function jsonResponse(payload) {
  return ContentService
    .createTextOutput(JSON.stringify(payload))
    .setMimeType(ContentService.MimeType.JSON);
}
```

## Deploy

1. Klik `Deploy`.
2. Pilih `New deployment`.
3. Tipe: `Web app`.
4. `Execute as`: akun Google kamu.
5. `Who has access`: `Anyone` atau `Anyone with the link`.
6. Copy URL Web App.

## Hubungkan ke dashboard

1. Buka panel `Google Calendar` di dashboard.
2. Paste URL Web App ke field webhook.
3. Klik `Simpan Webhook`.
4. Klik `Sync Semua Izin Valid` atau tombol `Sync` pada masing-masing izin.

## Catatan

- Event dibuat sebagai all-day event.
- Dashboard hanya mengirim event yang tanggal pengingatnya belum lewat.
- Jika ingin update event lama, marker `DASHBOARD-MINERBA:<event_key>` dipakai untuk mengenali event yang sudah pernah dibuat.
