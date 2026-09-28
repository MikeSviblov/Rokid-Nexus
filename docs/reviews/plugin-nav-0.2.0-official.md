# Ревью официального Navigation 0.2.0 (2026-09-28) — установлен на Pixel

Ревьюер — агент Claude; установка и проверка на Pixel 7 Pro — 2026-09-28. Номера строк — тег `nav-v0.2.0` (`49128717`),
`N/` = `plugins/nav/src/main/java/com/anezium/rokidbus/plugin/nav/`.

## Вердикт: ставить с условиями — установлен

| Что | Значение |
|---|---|
| URL | `https://github.com/Anezium/Rokid-Nexus/releases/download/nav-v0.2.0/nav-phone-release.apk` |
| SHA-256 APK | `6b4055c6721f593b243932bc489dad580d38bdf8f60ed8b5457ddf152911b36b` — = каталог RokidBrew-Registry, release notes, digest GitHub |
| Package / версия | `com.anezium.rokidbus.plugin.nav`, versionCode 3, 0.2.0 |
| Подписант | `CN=Anezium, OU=Rokid Nexus`, v2, RSA 4096 |
| **SHA-256 сертификата автора** | **`f5e938e2e79b0526b31e40d36c8c19098450c1636b7e14a306681b4effddf81c`** (тот же у всех плагинов Anezium и glasses-hub 1.5.0) |
| Воспроизводимость | сборка из `git archive nav-v0.2.0` совпала с официальным APK в 156 из 157 файлов, включая `classes.dex` и манифест; отличается только `version-control-info` |

**Манифест:** нет INTERNET, геолокации, контактов, SMS, хранилища, QUERY_ALL_PACKAGES, спецвозможностей. `allowBackup=false`.
NotificationListener защищён `BIND_NOTIFICATION_LISTENER_SERVICE`, `default_filter_types="ongoing|alerting|silent"`.
Capabilities в хабе — только `surfaces`. Ни один класс `com/anezium/*` не зовёт сеть (проверено по dexdump).

**Diff 0.1.0 → 0.2.0:** только `plugins/nav`; источники (точное имя пакета, `N/NavModels.kt:4–17`): Google Maps, Citymapper,
Organic Maps (`app.organicmaps`, `.web`), OsmAnd (`net.osmand`, `.plus`), Yandex Maps, maps.me. Фильтр пакета — до чтения
уведомления (`N/NavNotificationListener.kt:87–91`). Хранение — только переключатели. Логи без текста маршрута.

## Находки
- **H1 / MEDIUM (архитектурное):** Android отдаёт слушателю все уведомления разрешённых типов; allowlist источников — в коде
  плагина. Будущий APK с той же подписью может его снять, согласие хаба сохранится. Мера — фильтр типов в Android (ниже).
- **M1 / LOW-MEDIUM:** доверие CI и GitHub-аккаунту автора (ключ в GitHub Secrets). Мера — ручные обновления по чек-листу.
- **L1:** источник определяется по имени пакета без проверки подписи → приложение с именем неиспользуемого источника может
  выводить свой текст на HUD (утечки нет). Мера — выключать неиспользуемые источники.
- **L2:** новые источники включены по умолчанию.
- **L3:** RemoteViews Citymapper/Yandex разворачиваются в процессе плагина — максимум DoS.
- **L4:** `CitymapperParser.kt:46` `toInt()` без защиты (с 0.1.0).

## Поправка по итогам установки (2026-09-28)
Условие ревью «оставить только Real-time» **ломает Organic Maps**: его навигационное уведомление ongoing/FGS, category
`navigation`, канал `NAVIGATION`, но **importance LOW** — Android относит его к «Silent», и плагин его не получает (на очках
пусто, в логе ничего). С «Real-time + Silent» карточка появилась. **Принято (Mike): Real-time + Silent**, Conversations и
Notifications выключены. Дополнительно можно исключить чувствительные приложения в «See all apps».

## Установлено на Pixel (2026-09-28)
Наша 0.1.0 удалена (другая подпись), поставлена официальная; подпись установленного пакета сверена на телефоне
(`f5e938e2…`). Доступ к уведомлениям: включён, типы Real-time + Silent. В хабе одобрено только `surfaces`.
Проверено: маршрут Organic Maps → карточка на очках.
Экран настроек типов открывается командой:
`adb shell am start -a android.settings.NOTIFICATION_LISTENER_DETAIL_SETTINGS --es android.provider.extra.NOTIFICATION_LISTENER_COMPONENT_NAME com.anezium.rokidbus.plugin.nav/com.anezium.rokidbus.plugin.nav.NavNotificationListener`

## Чек-лист следующих обновлений
1. SHA-256 APK = release notes и запись `nav` в `nexus-plugins.v1.json`.
2. `apksigner verify --print-certs`: сертификат `f5e938e2…ddf81c`. Сменился — стоп.
3. `META-INF/version-control-info.textproto`: revision = коммит тега, тег в `upstream/main`.
4. Манифест: не появились INTERNET/геолокация/контакты/SMS/QUERY_ALL_PACKAGES/спецвозможности/overlay, новые exported,
   `<queries>`; CAPABILITIES только `surfaces`; `default_filter_types` не расширился.
5. `git diff <старый тег> <новый> -- plugins/nav bus-client shared ink-engine '*.gradle.kts' gradle .github`: список
   пакетов и точное сравнение в `NavModels.kt`, фильтр до `read` в `ingest`, что копирует `NavNotificationReader`, логи,
   зависимости.
6. Воспроизводимость: `git archive <тег>` → `./gradlew --offline :plugin-nav:assembleRelease -PskipCxrGlobal=true`
   **с очищенным окружением** (`env -u NEXUS_RELEASE_KEYSTORE -u NEXUS_RELEASE_KEYSTORE_PASSWORD -u NEXUS_RELEASE_KEY_ALIAS`,
   иначе сборка подпишется нашим ключом) → `classes.dex` и манифест совпадают с официальным.
7. Строки/сеть в dex: новые хосты или `java.net`/WebView из `com/anezium/*` — углублённое ревью.
8. `adb install -r` поверх; после — проверить, что в доступе к уведомлениям по-прежнему только Real-time + Silent.
