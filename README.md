# Noor – Namaz Reminder

Native Android (Kotlin + Jetpack Compose) prayer reminder for Muslims in Bangladesh.
Min SDK 26 (Android 8.0) · Target/Compile SDK 35 · No Room, no Retrofit: DataStore + `HttpURLConnection` keep it small and private.

## Open & run
1. Android Studio **Ladybug (2024.2)** or newer, JDK 17 (bundled).
2. *File ▸ Open* the `Noor` folder, let Gradle sync (needs internet the first time).
3. Run on a device or emulator (API 26+). For alarm/Azan testing a **real device** is best.

## Add the Azan audio (placeholder)
Put a properly licensed full-length file at `app/src/main/assets/azan/azan_default.mp3`
(any file named `azan_default.*`). Without it the app falls back to the phone's alarm tone so you can still test scheduling.
Users can also choose their own local file in *Settings ▸ Azan audio*. Playback is always local: never streamed.

## Project structure
```
Noor/
├── settings.gradle.kts · build.gradle.kts · gradle.properties
└── app/
    ├── build.gradle.kts · proguard-rules.pro
    └── src/main/
        ├── AndroidManifest.xml
        ├── assets/azan/            ← drop azan_default.mp3 here
        ├── res/                    (strings, colors, theme, launcher + notification icons)
        └── java/com/noor/namaz/
            ├── Models.kt           prayers, cities, methods, settings, time/Hijri formatting (Asia/Dhaka)
            ├── Repository.kt       DataStore (settings, records, offline cache) + AlAdhan API
            ├── Scheduler.kt        AlarmManager: prayer alarms, reminders, nightly refresh, reschedule-all
            ├── Receivers.kt        AlarmReceiver (prayer/reminder/done/snooze/refresh), BootReceiver
            ├── AzanService.kt      foreground media service, local audio, audio focus, wake lock
            ├── AzanActivity.kt     full-screen prayer screen over the lock screen
            ├── Notifs.kt · Perms.kt · NoorApp.kt (channels + WorkManager safety net)
            ├── MainActivity.kt     4-tab scaffold
            └── ui/                 Theme, Art (pattern/crescent/mosque), Home, TimesScreen (+Qibla),
                                    History, SettingsScreen, Onboarding, NoorViewModel
```

## How scheduling works
* `Scheduler.rescheduleAll()` cancels old alarms and sets exact alarms (`setExactAndAllowWhileIdle`) for the next ≤10 prayers
  from cached times (+ manual adjustments). It runs on: app start, any setting change, city/method change, new day,
  boot, app update, time/timezone/date change and exact-alarm permission change.
* At prayer time `AlarmReceiver` starts `AzanService` (foreground, `mediaPlayback`), which shows the full-screen intent,
  plays the Azan, then leaves a "pending prayer" notification and arms the repeating reminder.
* Reminders repeat every N minutes (default 5) until: **Done** is tapped, the prayer is marked completed/missed in the app,
  or the next prayer begins (each reminder carries its period's end time). Snooze = 15 min. Disable/adjust in Settings.
* Prayers are **never** auto-completed. Past pending prayers stay "pending" until you mark them.
* A nightly alarm (~00:10 Dhaka) and a 12-hourly WorkManager job fetch the next 3 days. Without internet the cache is used.
* If exact alarms are denied, Noor falls back to inexact Doze-friendly alarms and the Settings screen shows how to grant it.
* If the phone was off and an alarm is >20 min late, it posts a reminder instead of blasting a late Azan.

## Permissions
| Permission | Why |
|---|---|
| `POST_NOTIFICATIONS` | prayer screen + reminders (runtime, Android 13+) |
| `SCHEDULE_EXACT_ALARM` | Azan at the exact minute (grant via Settings ▸ Alarms & reminders on Android 12+) |
| `USE_FULL_SCREEN_INTENT` | full-screen prayer screen (Android 14+ may require enabling in app settings) |
| `FOREGROUND_SERVICE(_MEDIA_PLAYBACK)`, `WAKE_LOCK` | play the Azan reliably |
| `RECEIVE_BOOT_COMPLETED` | restore alarms after reboot |
| `INTERNET`, `ACCESS_NETWORK_STATE` | AlAdhan API |

No location permission: the city list has fixed coordinates (also used for Qibla). No analytics, no accounts, `allowBackup=false`.

## Manufacturer battery restrictions
Xiaomi/Redmi/POCO, Oppo/Realme/OnePlus, Vivo, Huawei/Honor, Samsung, Tecno/Infinix/itel may kill background apps.
Onboarding and *Settings ▸ Reliability & permissions* show step-by-step instructions for the detected brand, open the battery
settings page, and link to dontkillmyapp.com. Do this once, otherwise the Azan can be silenced.

## Testing tips
* *Settings ▸ Test Azan / Test notification*.
* Add +1 to a prayer's adjustment to move it a minute ahead of now, lock the phone and wait.
* `adb shell dumpsys alarm | grep -A3 com.noor.namaz` lists scheduled alarms.
* Doze: `adb shell dumpsys deviceidle force-idle`.

## Notes / design decisions
* API: `GET https://api.aladhan.com/v1/timingsByCity/{dd-MM-yyyy}?city=…&country=Bangladesh&method=1&school=0&timezonestring=Asia/Dhaka`.
  Asr uses the Standard school by default; a Settings toggle switches to Hanafi (common in Bangladesh).
* Hijri date comes from AlAdhan (cached); offline it falls back to the Umm al-Qura calendar. Bangladesh moon-sighting can differ by a day.
* Typography uses the system serif. To use Amiri/Scheherazade, add the font to `res/font/` and swap `FontFamily.Serif`.
* Google Play: full-screen intents and exact alarms are policy-sensitive; check current Play requirements before publishing.
