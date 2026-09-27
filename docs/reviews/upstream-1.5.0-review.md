# Ревью безопасности upstream Rokid Nexus 1.5.0

Дата: 2026-09-27. **Вердикт: можно с условиями.** Новых блокирующих регрессий в просмотренном обновлении не обнаружено. Это разрешает подготовку подписанных артефактов, но не означает устранение старых рисков или одобрение работы с секретами.

## Ревизии и метод

- Наш исходный `main`: `58e65578afbccc1bdd6ce8e56b845fbd16cc2576`.
- Последний общий upstream: `6a38c095101d241c682bf5d8f4dba4cdca3efbbb`.
- Проверенный `upstream/main`, peeled commit тега `v1.5.0`: `91f2662431b9251f185ea33a487a600904fec5f2`.
- `git diff main upstream/main`: 139 файлов, +10228/-1089 строк; `git rev-list --count main..upstream/main`: 79 коммитов. Двухточечный diff показывает отсутствие наших аудитных файлов и `.gitleaksignore` в upstream; это не их удаление при merge.
- Прочитан весь `AUDIT-2026-09-24.md`, включая H1–I1, чек-лист и добавленное подтверждение Bluetooth-подключения на устройстве. Старый эксперимент здесь не повторялся.
- Отдельные проходы: Navigation/регистрация, IME/Surface/Accessibility, транспорт/SelfArm/установщики, зависимости/workflows/общий diff. Выводы основаны на исходниках, а не changelog.
- Ссылки `file:line` ниже относятся к `91f26624` и совпадают с объединёнными исходниками. Сокращения: `G/` = `glasses-hub/src/main/java/com/anezium/rokidbus/glasses/`; `P/` = `phone-hub/src/main/java/com/anezium/rokidbus/phone/`; `S/` = `shared/src/main/java/com/anezium/rokidbus/shared/`; `N/` = `plugins/nav/src/main/java/com/anezium/rokidbus/plugin/nav/`; `C/` = `bus-client/src/main/java/com/anezium/rokidbus/client/`.

## Условия вердикта

1. Использовать проверенные release APK нашей подписи. Не использовать встроенную загрузку/установку glasses-hub, Store и upstream self-update для обновления этого набора: независимого signer pin для очкового APK всё ещё нет (H2). Эти пути в коде **не отключены**, а M1 позволяет локальному приложению инициировать некоторые административные действия; поэтому условие предполагает доверенный состав приложений телефона, не является техническим исправлением.
2. Не вводить пароли/личные заметки и другой чувствительный текст через Nexus или аппаратную клавиатуру при активном Accessibility до устранения M3 и проверки CXR. Требование старого AUDIT убрать keycode-логи для конфиденциального использования остаётся невыполненным. Для такого режима вердикт остаётся «нельзя»; условный допуск относится к ограниченному использованию без чувствительного ввода.
3. Не считать выключение Accessibility устойчивым отзывом полномочий (M2), а выключение Wireless ADB — удалением paired hosts. Pairing выдавать только доверенному компьютеру в доверенной сети; процессный timeout не гарантирует закрытие native сервиса при сбое (M4).
4. Не удалять/переустанавливать glasses-hub под другим ключом, полагаясь на остановку старого shell-helper (M5). Установка и disarm в этой задаче не выполнялись.
5. Navigation требует отдельного Android Notification Access и только Nexus `surfaces`. Доступ ОС шире двух источников; выдавать его осознанно. Включение Navigation/Maps/Citymapper по умолчанию не выдаёт системное разрешение автоматически. Чувствительный маршрут также может пройти через CXR fallback; собственного сквозного шифрования нет.
6. Проверить новый переключатель удержания Nexus keyboard: по умолчанию он включён и возвращает Nexus вместо `com.rokid.*` IME. Если это не нужно, отключить его. `SYSTEM_ALERT_WINDOW` телефону для ручной клавиатуры не требуется; без разрешения используется уведомление.

## Известные находки: повторная проверка

| ID / severity | Состояние в 1.5.0 и доказательства |
|---|---|
| **H1 / HIGH в исходном AUDIT** | Исходный обход неаутентифицированным SPP закрыт уже в нашем baseline и не восстановлен обновлением. Insecure RFCOMM остаётся (`G/SppServerManager.kt:89`), но вход проходит mutual proof и отдельную authenticated session до допуска к bus/выходу: `G/AuthenticatedSppServer.kt:44`, `:120`; `S/SppAuthProtocol.kt:35`, `:112`. Pending peer не получает общий output, существующий active peer не заменяется. MAC и последовательность защищают целостность/replay; это **не шифрование**. CXR остаётся корнем доверия enrollment и fallback; см. ниже. |
| **H2 / HIGH** | **Не исправлено.** `P/GlassesApkVerificationPolicy.kt:22` принимает package без сертификата, `:33` допускает digest fallback при неразобранном APK на старом API. `P/BusHubService.kt:4085` digest опционален, `:4089` policy, `:4131` CXR install. Собственная подпись артефактов не исправляет установщик. |
| **M1 / MEDIUM** | **Не исправлено.** `phone-hub/src/main/AndroidManifest.xml:150` экспортирует BusHubService без permission; `P/BusHubService.kt:950`–`:1005` принимает STOP/SET_TOKEN/install/open/setup start-intent без проверки отправителя. Ограничения Android background execution и наличие соединения влияют на достижимость. Новый Binder keyboard route, напротив, internal-only. |
| **M2 / MEDIUM** | **Не исправлено.** `G/AccessibilityRearmWatcher.kt:71`–`:80` и `glasses-hub/src/main/assets/rokid-nexus-a11y-watchdog.sh:140`, `:191` восстанавливают Accessibility. Opt-out нового keyboard keeper не является Accessibility disarm. |
| **M3 / MEDIUM** | **Не исправлено.** `G/RokidBusAccessibilityService.kt:169`–`:175` пишет аппаратные keycode/action/repeat/time; `G/Logx.kt:7`–`:8` вызывает release Log.i. Подавление только для focused Nexus editable Surface не защищает стороннее password/text поле. Не утверждается, что логируется каждый символ soft IME. |
| **M4 / MEDIUM** | **Не исправлено.** `G/WirelessAdbController.kt:186`–`:228`: process executor, ограниченные retry, после отказов `clearPairingState` с отказом от дальнейших попыток. `:240` disable не отзывает paired hosts. Native timeout после смерти процесса не проверен. |
| **M5 / MEDIUM** | **Не исправлено.** `glasses-hub/src/main/assets/rokid-nexus-cmd-bridge.sh:504`, `:518`, `:538`: helper извлекает и запускает новый script по package/version, без независимого signer check. Условие эксплуатации — переживший uninstall helper и повторная установка чужого пакета; на устройстве не проверено. |
| **L1 / LOW** | **Не исправлено.** `plugin-feeds/src/main/java/com/anezium/rokidbus/plugin/feeds/FeedsSettingsStore.kt:66`, `:81` — обычные prefs; `P/BusHubService.kt:968` — token prefs; `plugins/assistant/src/main/java/com/anezium/rokidbus/plugin/assistant/AssistantThreadStore.kt:49`, `:290` — private JSON/JPEG. Android sandbox сохраняется; это не world-readable storage. Новые Assistant prefs — режимы ввода/визуализации, а не новые plaintext ключи. |
| **L2 / LOW** | **Не исправлено.** `plugins/lens/src/main/java/com/anezium/rokidbus/plugin/lens/OnlineTranslationProviders.kt:384`–`:387` выводит `response.take(120)` при parse failure. Lens не входит в четыре собираемых APK. |
| **I1 / INFO** | **Сохраняется.** `settings.gradle.kts:3`, `:13`: mavenLocal раньше удалённых; `:16`, `:17`: JitPack/Rokid Maven. `gradle/wrapper/gradle-wrapper.properties:3`: wrapper 9.5.1 без distributionSha256Sum. Tracked locks/verification metadata не найдены; vendor binaries не удостоверены. Теперь CxrGlobal присутствует, точная ревизия зафиксирована ниже. |

## Navigation: источники, данные, границы

`N/NavModels.kt:4`–`:10` принимает только `com.google.android.apps.maps` и `com.citymapper.app.release`. `N/NavNotificationListener.kt:86`–`:92` отбрасывает остальные packages и выключенные источники **до** чтения содержимого. Maps требует ongoing/navigation (`N/GoogleMapsParser.kt:41`), Citymapper — ongoing и подходящий channel, допуская null (`N/CitymapperParser.kt:29`). Это прикладной фильтр, не ограничение полномочий NotificationListener на уровне ОС.

`N/NavNotificationReader.kt:16`–`:37` читает title/text/subText/bigText/textLines, critical/now-bar extras, progress и названия actions. Для Citymapper `:46`–`:60` применяется RemoteViews и собирается текст видимых TextView по resource id. PendingIntent уведомления не исполняется; screenshot чужого приложения, accessibility tree и location provider не используются. Это доверие содержимому двух установленных приложений, а не криптографическая проверка их издателя.

Поток: уведомление → parser → `NavGuidance` в RAM → `N/NavRuntime.kt:153`–`:175` → SDK `/activity/start|update|end` и собственная `/surface` card → phone-hub → SPP либо CXR → glasses HUD. Передаются следующий поворот/остановка/улица, расстояние/время/ETA, линия, прогресс/track/measure (`N/NavRuntime.kt:197`–`:210`). Локально маршрут также виден в settings/card (`N/NavSettingsActivity.kt:87`, `N/NavCard.kt:19`). Географически значимые данные покидают процесс плагина и телефон ради отображения на очках; формулировка «никуда не отправляет» была бы неверна.

В исходниках Navigation, shared и bus-client не найдены собственные HTTP/WebSocket/Socket вызовы; в их source manifests нет INTERNET. Navigation не запрашивает `http_proxy`. Существующие внешние запросы phone-hub (Store/releases и настроенный speech) продолжают существовать независимо от Navigation. Новых прикладных сетевых endpoints в Kotlin/Java diff не найдено.

`N/NavState.kt:7`–`:16` держит route в RAM; `N/NavRuntime.kt:50`–`:75` очищает при окончании/отключении. `N/NavSettings.kt:23`–`:40` сохраняет только switches. Логи listener (`:55`, `:63`, `:83`, `:105`) и runtime (`:175`, `:187`) содержат source/category/channel/glyph/result/reason, не текст маршрута. Channel id — метаданные источника, не обещание полной анонимности.

Manifest: `plugins/nav/src/main/AndroidManifest.xml:4`–`:6` — REQUEST_DELETE_PACKAGES, FOREGROUND_SERVICE, FOREGROUND_SERVICE_SPECIAL_USE; `:15` allowBackup=false. Settings Activity exported (`:22`), но не принимает административные extras; действия через UI. NotificationListener exported (`:29`), защищён системным BIND_NOTIFICATION_LISTENER_SERVICE (`:33`). Plugin service exported (`:42`) для SDK, требует Nexus grant только `surfaces` (`:73`), receive prefixes `/plugin/nav,/system/plugin` (`:76`). Произвольного externally supplied bus payload через start-intent не обнаружено.

**N1 / INFO — широкий системный Notification Access.** Несмотря на кодовый allowlist, разрешение относится к listener целиком. Отказ в Nexus grant не отзывает Android Notification Access. Это новая явно принимаемая поверхность доверия, а не найденная массовая выгрузка уведомлений. Семантика сервиса сверена с [Android NotificationListenerService](https://developer.android.com/reference/android/service/notification/NotificationListenerService).

**N4 / LOW — некорректное число остановок может уронить Navigation listener.** `N/CitymapperParser.kt:40`–`:41` вызывает `toInt()` на совпавших цифрах без безопасного преобразования. Например, `2147483648 stops` в ожидаемом title layout приводит к NumberFormatException; обычный posted callback не перехватывает исключение парсера. Это статически выявленный риск устойчивости при неожиданном содержимом Citymapper, не доказанный доступный любому приложению DoS: package allowlist сохраняется. Parser fuzzing/PoC на устройстве не выполнялся. Для четырёх APK не признан security-blocker; будущая правка — toIntOrNull с отказом на невалидном числе.

## IME, прозрачный Surface, права и шина

**N2 / LOW — удержание Nexus IME включено по умолчанию.** `S/GlassesKeyboardContract.kt:20` задаёт default keep; `G/GlassesKeyboardKeeper.kt:30`, `:57`, `:65`, `:89` применяет его при старте и смене DEFAULT_INPUT_METHOD. Возврат только из пустого выбора или package `com.rokid.*`; уже выбранные сторонние IME не заменяются. Сохраняемый выключатель есть; бюджет — три успешных автоматических возврата за десять минут. Отдельное действие select может быть выполнено владельцем. Это расширяет активность IME относительно AUDIT, но не добавляет чтение уже существующего текста редакторов.

**N3 / INFO — новое разрешение SYSTEM_ALERT_WINDOW на телефоне.** `phone-hub/src/main/AndroidManifest.xml:27`; `P/RemoteKeyboardPrompt.kt:47`–`:58` открывает RemoteInputActivity только при выданном overlay grant и unlocked screen, иначе показывает уведомление. PendingIntent immutable (`:83`–`:89`), его текст фиксированный, без editor content. `P/RemoteInputActivity.kt:418` открывает системные настройки по кнопке. Разрешение не выдаётся автоматически; оно шире нужды ручного ввода. См. [Android SYSTEM_ALERT_WINDOW](https://developer.android.com/reference/android/Manifest.permission#SYSTEM_ALERT_WINDOW).

SurfaceActivity остаётся exported=false, singleTask, excludeFromRecents; изменены taskAffinity и translucent theme (`glasses-hub/src/main/AndroidManifest.xml:80`, `glasses-hub/src/main/res/values/styles.xml:19`). Чужое приложение под ним остаётся видимым, но новый код не читает его содержимое. `G/NoticeComposeMirror.kt:60` связывает зеркало текста с ownerPluginId видимой notice, `G/SurfaceHudView.kt:529` копирует только своё EditText в локальную RAM; очистка предусмотрена при смене owner/detach. `G/HudIsland.kt:149` делает bitmap **собственного View** для crossfade, не screenshot экрана, без записи файла/сети, recycle при detach/конце (`:196`, `:291`).

Accessibility XML и основные event/key handlers не изменены: сохраняются typeAllMask/window content/global package scope, новых package/event scopes нет. Прямой утечки нового вводимого текста в прикладные логи в diff не найдено; M3 остаётся. RemoteInput сохраняет session/sequence gate и лимиты (`G/RemoteInputSession.kt:57`, `:76`). Новая отметка keyboardRequested сообщает, что поле просит телефонную клавиатуру; не переносит editor text. Она принимается только от собственного editor package с точной privateImeOptions (`G/RemoteInputSession.kt:115`–`:116`); произвольный сторонний редактор не может активировать её одним совпадением privateImeOptions. Защита password UI телефона и запрет сохранения editor view-state сохранены. Реальное поведение transparent task, focus и secure window на Rokid не проверялось.

Новые `/glasses/keyboard/request|reply` — trusted control, не plugin capability. `P/BusHubService.kt:1106`–`:1111` требует UID самого хаба до обработки; `:1886`–`:1894` классифицирует keyboard request как internal. `PluginRoutePolicy`/`PathRules` продолжают запрещать reserved namespaces; `/core/native-apps/*`, `/core/remote-input/*`, `/core/navigation/*`, `/core/pointer/*` не выданы плагинам. PluginCapability enum и receive allowlist не расширены. DebugLegacy не стал доступен release.

`C/plugin/NexusPluginClient.kt:76`, `:494`: registrationGeneration — локальный счётчик с private setter, меняется при registration callback, не токен полномочий и не wire principal. Navigation сравнивает его, чтобы после reconnect заново отправить activity; повторное approval metadata не должно создавать вторую activity. Серверная UID/package/signer/grant проверка сохранена.

Activity extras — badge/measure/track/urgent — проходят общий validator. `S/ActivitySurfaceContract.kt:146`: badge ≤5, measure ≤8, track count 2..12, label ≤20; `:527` проверяет целые индексы и at≤target<count; `:334` urgent требует significant и разрешён только update. Renderer работает с bounded typed data. Это feature negotiation (`ACTIVITY_EXTRAS`), не новый grant; `surfaces` и owner scoping остались. Недоверенный pluginId не берётся из тела нового поля.

Assistant/Relay получили inline input, локальные режимы и ограничение визуальных ответов; отдельного нового сетевого транспорта/секретного storage в diff не найдено. Введённый Assistant prompt по назначению может попасть к настроенному AI provider, Relay reply — в messaging app; это не offline-ввод. Они не входят в собираемый набор.

**N5 / LOW — Sample расширяет доступный извне demo-control.** `plugins/sample/src/main/AndroidManifest.xml:29`–`:32` экспортирует HelloPluginService без permission. Новый `plugins/sample/src/main/java/com/anezium/rokidbus/plugin/sample/HelloPluginService.kt:76`–`:83` принимает ACTION_DEMO_ACTIVITY/step без UID/DEBUG gate; `:126`–`:147` запускает фиксированные HUD-сценарии, `:634`–`:645` допускает wakeDisplay и длительность 30 минут. Другое приложение, способное запустить сервис с учётом Android ограничений, может навязать или остановить canned activity от имени ранее одобренного Sample. Это не обход выдачи surfaces grant, не произвольный текст/shell/чтение данных. Старый DEMO_NOTICE уже имел такой класс проблемы; обновление расширяет его. Sample отсутствует в четырёх APK, поэтому находка не блокирует этот набор. Runtime exploit не выполнялся.

## H1, диагностика SPP, привилегированный bridge

В сравнении main→upstream auth-код SPP меняется только в диагностике. `G/AuthenticatedSppServer.kt:93`: `SPP TX <path> id=<id>` после успешной записи; `:96` — только simpleName исключения. Ни payload, ни exception.message/body, ни pairing code сюда не добавлены. Новый `AuthenticatedSppServerTest.kt:179` проверяет отсутствие тестового secret payload в логе. Path/id — управляемые метаданные; это не универсальный secret scrubber для произвольного содержимого id, но штатный Wireless ADB не кодирует pairing code в них.

`S/SppAuthProtocol.kt:112`–`:150` проверяет размер/MAC/sequence; новая транспортная сессия не принимает replay предыдущей. Enrollment ключа через доверяемый по модели CXR (`G/CxrBusBridge.kt:98`, `S/SppKeyProvisioning.kt:37`); доступность и конфиденциальность vendor path не доказаны статикой. Inner JSON при CXR fallback остаётся открытым для vendor (`P/BusHubService.kt:3240`, `G/GlassesOutboundTransportPolicy.kt:38`). Комментарии о vendor logging не считаются нашим измерением.

SelfArm/WirelessAdb/shell assets не менялись в этом обновлении и ключевые границы перечитаны: фиксированный набор действий, API32 guards, проверенные аргументы вместо нового произвольного shell, nonce/replay/digest bridge сохранены. `/debug/adb/request` требует high-risk wireless_debugging, phone-hub ставит проверенный pluginId, reply owner-scoped; pairing code остаётся в RAM/ответе, не в prefs. FLAG_SECURE экрана и clipboard только по явному copy сохранены. M2/M4/M5 при этом остаются.

## Сборочная цепочка

Изменение Gradle: версии хабов 1.5.0/10500, версии изменённых плагинов и добавление `:plugin-nav` → `plugins/nav`. У Navigation локальный bus-client и тестовый JUnit 4.13.2, общий signing script. Новых внешних repositories, dynamic dependency versions, Gradle plugins или signing hooks в diff нет. `.github/workflows/plugin-release.yml:89` добавляет nav в существующий release selector; app-release workflow неизменён. Workflow actions по major tags, не immutable SHAs, — сохранённое ограничение supply chain.

CxrGlobal: `/home/mike/projects/CxrGlobal`, чистый checkout `bfc3fab18268d6dd4b2c371416189cb19610b631`, tag `v0.2.3`. Совпадает с `.github/workflows/app-release.yml:23`–`:28`; новая ревизия для 1.5.0 не требуется по diff и результату успешной сборки. `settings.gradle.kts:56` подключает sibling composite, `../CxrGlobal/lib/build.gradle.kts:9` version 0.2.0 — координата не равна release tag; фактический vendor `client-l:1.1.0` объявлен в `:33`. CxrGlobal checkout, конфигурация SDK/Gradle и local.properties не менялись; использованы обычные системные кэши сборки.

Tracked APK/AAR/SO/DEX/keystore отсутствуют; tracked исполняемый архив — прежний gradle-wrapper.jar. Зависимости из Maven и native/vendor binary contents этим не удостоверены. Wrapper JAR checksum по эталону, CVE-скан и воспроизводимость бинарников не проверялись. Вызовы/логи CxrGlobal Kotlin просмотрены выборочно; native CXR/HiRokid остаются вне доказанного контура.

## Merge и проверка артефактов

Создана ветка `update/1.5.0` от исходного main. Merge commit: `85beeb5f61b1ffbbe948aba36840d152df844f6f` (parents: исходный main и 91f26624). Merge выполнен ort **без конфликтов**, ручных разрешений нет. Сохранены ровно три fork-only файла/изменения: `.gitleaksignore` (13 строк), `AUDIT-2026-09-24.md`, `security-report-H1.md`. Код merge совпадает с upstream. Main не передвигался.

### Тесты и сборка

Выполнено без `-PskipCxrGlobal=true`:

```bash
./gradlew :shared:testDebugUnitTest :phone-hub:testDebugUnitTest :glasses-hub:testDebugUnitTest :plugin-wireless-adb:testDebugUnitTest :plugin-nav:testDebugUnitTest :bus-client:testDebugUnitTest
./gradlew :phone-hub:assembleRelease :glasses-hub:assembleRelease :plugin-nav:assembleRelease :plugin-wireless-adb:assembleRelease
```

SDK suite добавлен из-за изменения registrationGeneration и activity API. По XML-результатам: shared 356, phone-hub 609, glasses-hub 678, Wireless ADB 7, Navigation 37, bus-client 131 — **1818 тестов, 0 failures/errors/skipped**. Часть Gradle tasks использовала cache/up-to-date; это не обещание чистой воспроизводимой сборки. В результаты входят transport authentication/negative cases и новый regression test TX logging. Процессные и аппаратные сценарии ниже остаются непроверенными.

Реальный хвост вывода unit-команды:

```text

BUILD SUCCESSFUL in 56s
159 actionable tasks: 83 executed, 3 from cache, 73 up-to-date
Consider enabling configuration cache to speed up this build: https://docs.gradle.org/9.5.1/userguide/configuration_cache_enabling.html
```

Реальный хвост вывода release-команды:

```text

BUILD SUCCESSFUL in 48s
302 actionable tasks: 122 executed, 19 from cache, 161 up-to-date
Consider enabling configuration cache to speed up this build: https://docs.gradle.org/9.5.1/userguide/configuration_cache_enabling.html
```

Сборка также выполнила lintVitalRelease для четырёх APK. Были warnings deprecated Android API/Gradle features и always-false condition в RemoteInputActivity:752; ошибок сборки нет. Фактический `./gradlew --version`: Gradle 9.5.1, Launcher/Daemon JVM **21.0.12.1 Debian**, `/usr/lib/jvm/java-21-openjdk-amd64`. Это отличается от рекомендации Java 17 в README/AGENTS; окружение не переключалось. Не следует приписывать эти результаты JDK 17.

### Подписи, версии и хэши

Для каждого APK выполнены `/home/mike/Android/Sdk/build-tools/36.0.0/apksigner verify --verbose --print-certs`, `aapt dump badging`, `sha256sum`. Все четыре: `Verifies`, APK Signature Scheme v2=true, один signer, RSA 4096, `CN=Mike Sviblov, O=green.irish`; debuggable отсутствует. SHA-256 сертификата совпадает между APK и с публичным сертификатом ключа из настроенного release keystore (сравнение через keytool exportcert; значения NEXUS_RELEASE_* не печатались):

```text
893070f80109bed8abebb458eee9df9e2631c03b56bf8d78d5a0818b26e92134
```

| APK (абсолютный путь) | package | versionName / versionCode | SHA-256 APK |
|---|---|---|---|
| `/home/mike/projects/rokid-nexus/phone-hub/build/outputs/apk/release/phone-hub-release.apk` | `com.anezium.rokidbus.phone` | 1.5.0 / 10500 | `578a5501e8418841d124f9f0fedd2477bf6e5938341ef48dd8371bea5e5b430e` |
| `/home/mike/projects/rokid-nexus/glasses-hub/build/outputs/apk/release/glasses-hub-release.apk` | `com.anezium.rokidbus.glasses` | 1.5.0 / 10500 | `5a46ebe90a0c8849ede158c21098654570729bd8ec9e523ef138643a8b8a44d4` |
| `/home/mike/projects/rokid-nexus/plugins/nav/build/outputs/apk/release/plugin-nav-release.apk` | `com.anezium.rokidbus.plugin.nav` | 0.1.0 / 1 | `fdac70ceb5e43090e701bd1556d855c6e2e0016cb4a414452fac27339c216385` |
| `/home/mike/projects/rokid-nexus/plugins/wireless-adb/build/outputs/apk/release/plugin-wireless-adb-release.apk` | `com.anezium.rokidbus.plugin.wirelessadb` | 1.0.1 / 2 | `18fc0fbce66e721623c8c497393977d0e7271481987a1d50e9b608b7a8dc89b8` |

Готовый Navigation APK не содержит INTERNET. Packaged release manifest дополнительно содержит AndroidX signature permission `com.anezium.rokidbus.plugin.nav.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION`, приватный InitializationProvider и системно защищённый ProfileInstallReceiver; новых незащищённых обработчиков содержимого уведомлений от библиотек не обнаружено. Запрошенные в исходниках delete/FGS permissions сохранены; BIND_NOTIFICATION_LISTENER_SERVICE по-прежнему стоит на listener, allowBackup=false.

Полные временные логи: `/tmp/rokid-nexus-1.5.0-tests.log`, `/tmp/rokid-nexus-1.5.0-release.log`; JSON проверки APK: `/tmp/rokid-nexus-1.5.0-artifacts.json`. Это вспомогательные локальные файлы, не часть коммита. Отчёт хранит ключевые результаты независимо от их срока жизни.

## Покрытие чек-листа AUDIT

| Пункт | Результат |
|---|---|
| Commits, CxrGlobal, release workflows | Ревизии зафиксированы; composite/CI tag совпадают; новая ревизия CxrGlobal не нужна. |
| SPP/CXR/FrameProtocol/ingress/ownership/replay/fallback/logs | H1-фикс сохранён; TX metadata-only; CXR confidentiality остаётся ограничением. |
| Paths/grants/principals/receive/DebugLegacy | Keyboard control internal-only; extras не новый grant; release legacy guard сохранён. |
| SelfArm/WirelessADB/shell/nonce/API guards/disarm | Diff отсутствует, ключевые guards перечитаны; M2/M4/M5 не исправлены. |
| Accessibility/RemoteInput/IME/clipboard/screenshot | Scope не расширен; keeper default изменён; локальное зеркало своего поля; M3 остаётся. |
| Все изменённые manifests и конечные APK | Navigation listener/экспорты, phone overlay permission, private translucent task; release/signatures/Navigation permissions проверены. |
| Registry/install/update URLs/digest/signers | Реализации не менялись; H2/M1 перечитаны и остаются; offline-флага нет. |
| HTTP/WS/Socket/WebView и динамические адреса | Поиск по изменённым прикладным исходникам и полная проверка нового Navigation: новых network callers/endpoints нет; прежние sinks из AUDIT сохраняются, runtime endpoints не измерены. |
| Storage/logs/secret lifecycle | Navigation RAM/boolean prefs; no payload TX; новые UI prefs; L1/L2/M3/M4 остаются. |
| Gradle/workflow/prebuilt/signing и negative verification | Source diff, тесты merge, четыре подписанных release APK; аппаратные негативные сценарии не выполнялись. |

Независимый reviewer подтвердил security-выводы и условия вердикта; замечания о точности line references и доверии CXR учтены. Отдельный verifier повторно проверил четыре APK через apksigner/aapt/sha256sum, сверил логи, XML-счётчики и merge parents; расхождений не обнаружено. Сопоставление с сертификатом самого keystore выполнено основным исполнителем, без повторного чтения ключа verifier-ом.


## Ограничения проверки

- Ни adb, ни установка/запуск на телефоне/очках, ни push/merge в main не выполнялись.
- Нет нового Bluetooth PoC, проверки чужого Android UID на устройстве, traffic capture, vendor logcat, kill/reboot во время pairing, runtime проверки opt-out/IME, wrong-signer установки или проверки прошивки не-API32. Unit tests не заменяют эти сценарии.
- Не доказана конфиденциальность CXR/vendor и радио; HMAC SPP не равен end-to-end encryption. Компрометированный доверенный хаб/телефон вне защиты плагиновых grants.
- Проверка внешних приложений Maps/Citymapper, произвольных notification layouts, всех firmware/Android версий, parser fuzzing и полного dependency CVE graph не проводилась. Неизвестные локализации/layouts могут давать неверную/отсутствующую подсказку.
- Статическое ревью security-sensitive diff не является полной повторной функциональной проверкой всех плагинов и сайта. Существующие Assistant/Agents/Tasker/облачные интеграции не получают нового общего одобрения.
- MemOS MCP в доступном наборе инструментов отсутствовал; кросс-проектная семантическая память не использовалась. Источники решения — репозиторий, исходный AUDIT и перечисленная официальная Android документация.
