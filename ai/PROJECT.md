# PROJECT.md

Техническая документация проекта WhiteListCheck. Меняется редко.

## Описание проекта

«Белый список?» — Android-приложение для диагностики режима «белого списка»
мобильного интернета в РФ. Определяет по доступности трёх групп сайтов,
что происходит с сетью: норма / белый список / нет интернета / VPN.

- Репозиторий: https://github.com/dmitrystarosta/WhiteListCheck
  (переименован из NetStatus; старые ссылки работают через редирект GitHub)
- Релизы: https://github.com/dmitrystarosta/WhiteListCheck/releases
- Лицензия: MIT, © 2026 Dmitry Starosta
- Название приложения на устройстве: «Белый список?»
- Имя APK-файла: WhiteListCheck.apk

## Архитектура

Одна Activity (MainActivity), UI на Jetpack Compose, весь код в одном
Kotlin-файле. Логические блоки внутри MainActivity.kt:

- `Probe`, `ProbeResult`, `Verdict`, `ScanState` — модель данных
- `ProbeConfig` — три встроенных списка сайтов (defaultA/B/C), константа
  REMOTE_CONFIG_URL и парсер JSON `{"a":[{"name","url"}],"b":[...],"c":[...]}`
  для будущего удалённого обновления списков (сейчас URL пуст)
- `Scanner` — object: параллельные HEAD-запросы (HttpURLConnection, таймаут
  4000 мс, редиректы выкл.; «доступен» = получен любой HTTP-код), перевод
  исключений в русские сообщения (humanError), определение типа сети через
  ConnectivityManager/NetworkCapabilities, вычисление вердикта по большинству
- `CheckWorker` (CoroutineWorker) — фоновая проверка: пропускает «нет сети»,
  сравнивает вердикт с сохранённым в SharedPreferences ("netstatus" /
  "last_verdict"), при изменении шлёт уведомление (канал "netmode", иконка
  R.drawable.ic_launcher, PendingIntent на MainActivity); на API 33+
  проверяет разрешение POST_NOTIFICATIONS
- `scheduleBackground` / `cancelBackground` — PeriodicWorkRequest 15 минут,
  constraint NetworkType.CONNECTED, уникальное имя "netcheck",
  ExistingPeriodicWorkPolicy.UPDATE
- Composable: `App` (состояние, кнопка, переключатель фона, запрос
  разрешения уведомлений через rememberLauncherForActivityResult),
  `VerdictCard`, `GroupHeader`, `ProbeRow` (кликабельное имя →
  LocalUriHandler.openUri на https://host), `Footnote` (объяснение ошибок +
  сноска про Instagram), `AppFooter` (версия из PackageManager + копирайт,
  ссылка на страницу релизов — константа REPO_RELEASES)

Ручная проверка тоже пишет вердикт в last_verdict, чтобы фон сравнивал
с актуальным состоянием.

## Структура каталогов

```
WhiteListCheck/
├── .github/workflows/build.yml
├── app/
│   ├── build.gradle.kts
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/ru/netstatus/app/MainActivity.kt
│       └── res/drawable/ic_launcher.xml
├── build.gradle.kts
├── settings.gradle.kts
├── gradle.properties
├── LICENSE
└── README.md
```

## Описание основных файлов

- **MainActivity.kt** — весь код приложения (см. Архитектура).
- **AndroidManifest.xml** — разрешения INTERNET, ACCESS_NETWORK_STATE,
  POST_NOTIFICATIONS; label «Белый список?»; icon @drawable/ic_launcher;
  theme @android:style/Theme.Material.Light.NoActionBar;
  usesCleartextTraffic="false".
- **ic_launcher.xml** — векторная иконка «чебурнет» (глобус с ушами):
  фон-скругление #4E342E, заливка #6D4C41, контуры #FFCCBC, viewport 220.
  Дизайн намеренно без узнаваемого персонажа (авторские права).
  Для магазинов есть растровая версия icon_512.png (генерируется из тех же
  координат).
- **app/build.gradle.kts** — versionCode/versionName; зависимости (Compose
  BOM, activity-compose, material3, coroutines, work-runtime-ktx);
  signingConfig "release", читающий keystore.properties из корня проекта
  (файл создаётся только в CI из секретов; в defaultConfig applicationId
  ru.netstatus.app); buildTypes.release: minify выкл., debuggable выкл.
- **gradle.properties** — android.useAndroidX=true, jvmargs, code style.
- **settings.gradle.kts** — репозитории, rootProject.name = "netstatus"
  (внутреннее имя, наружу не видно, менять не обязательно).
- **README.md** — описание, раздел «Когда пригодится» с ключевыми фразами
  для поиска, таблица групп, вердикты, установка (Play Protect), сборка,
  планы, сноска про Instagram.

## Принцип работы приложения

1. Пользователь жмёт «Проверить» (или срабатывает фоновый воркер).
2. Если задан REMOTE_CONFIG_URL — попытка скачать свежие списки (не критично).
3. Параллельные HEAD-запросы ко всем сайтам трёх групп.
4. Подсчёт «большинства живых» в каждой группе → вердикт → цветная карточка.
5. Фон: вердикт сравнивается с прошлым; отличие → push-уведомление.

## Процесс сборки

Локальная сборка не используется. Всё собирает GitHub Actions на каждый
push в main (плюс ручной запуск workflow_dispatch). Готовый APK — в
Artifacts сборки. Релиз оформляется вручную через GitHub Releases
(тег vX.Y, приложить APK, Set as the latest release).

## GitHub Actions (.github/workflows/build.yml)

Шаги (целевое состояние с release-подписью):
1. checkout, JDK 17 (temurin)
2. Prepare keystore: `echo "$KEYSTORE_BASE64" | base64 -d > release.jks`;
   генерация keystore.properties (storeFile=../release.jks + пароли/алиас
   из секретов)
3. Build: если нет gradlew — `gradle wrapper --gradle-version 8.7`;
   `./gradlew assembleRelease`
4. Rename: app/build/outputs/apk/release/app-release.apk → WhiteListCheck.apk
5. upload-artifact: name WhiteListCheck, path WhiteListCheck.apk

Секреты (Settings → Secrets and variables → Actions):
KEYSTORE_BASE64, KEYSTORE_PASSWORD, KEY_ALIAS, KEY_PASSWORD.

## Подпись APK

- Release-ключ: локально у владельца C:\AndroidKeys\whitelistcheck-release.jks,
  алиас whitelistcheck, RSA 4096, validity 10000 дней; копия файла и пароль —
  в облаке владельца. Пароль ключа = паролю хранилища.
- Ключ сгенерирован keytool из Android Studio JBR через PowerShell.
- В CI ключ восстанавливается из секрета KEYSTORE_BASE64.
- История: первый ключ скомпрометирован (keystore.properties с паролями
  закоммичен в публичный репозиторий) и заменён. Правило: ключи и пароли
  в репозиторий не попадают никогда; помнить, что git хранит историю.
- Потеря ключа = невозможность обновлять приложение в магазине.

## Особенности публикации в RuStore

- Аккаунт физлица: бесплатно, вход через VK ID, верификация по паспорту
  (фото + лицо), без УКЭП. Аккаунт Дмитрия зарегистрирован и верифицирован.
- Требуется: release-подписанный APK, иконка 512×512, минимум 2 скриншота,
  краткое+полное описание, ссылка на политику конфиденциальности,
  категория (Инструменты), возраст (0+).
- Скриншоты для карточки подготовлены БЕЗ упоминания Instagram (обрезаны
  выше блока «Заблокированные в РФ») — чтобы не смущать модерацию.
- Консоль: console.rustore.ru. Модерация — от часов до пары дней.

## Используемые технологии

Kotlin 1.9.24 · Jetpack Compose (BOM 2024.06.00, Material3) · WorkManager
2.9.0 · kotlinx-coroutines 1.8.1 · HttpURLConnection · AGP 8.4.0 · Gradle 8.7
· JDK 17 · compileSdk/targetSdk 34, minSdk 26 · GitHub Actions ubuntu-latest.

## История принятых технических решений

- **Однофайловый код** — осознанно: владелец правит файлы через веб-GitHub,
  один файл проще заменять целиком.
- **HttpURLConnection вместо OkHttp** — ноль лишних зависимостей.
- **HEAD к favicon.ico, «жив = любой HTTP-ответ»** — отличает реальную
  доступность (TLS + ответ) от DNS-заглушек и сбросов соединения.
- **Вердикт по большинству** — один лежащий сайт не даёт ложной тревоги.
- **Тема @android:style/...** — библиотека тем Material3 не подключена;
  две сборки падали на Theme.Material3.DayNight.NoActionBar, зафиксировано
  как ограничение.
- **gradle.properties добавлен после падения** checkDebugAarMetadata
  (android.useAndroidX=true).
- **Имя пакета ru.netstatus.app сохранено** при переименовании репозитория
  NetStatus → WhiteListCheck: смена пакета = другое приложение для Android.
- **Иконка вектором в XML** — рисуется текстом через веб-редактор, весит
  <1 КБ, масштабируется; «чебурнет» без прямого образа Чебурашки.
- **Ошибки по-русски + сноска** — вместо имён исключений; три разных
  механизма блокировок наглядно различимы.
- **МосКостюмер добавлен в группу B, затем удалён** (v0.2) — слишком
  нишевый сайт, вызывал бы вопросы посторонних пользователей.
- **Секреты GitHub + сборка assembleRelease в CI** — после инцидента
  с утечкой паролей; ключ в репозиторий не попадает даже в base64
  (только в Secrets).
- **Google Play отложен**: оплата $25 из РФ затруднена, для личных
  аккаунтов нужны 12 тестировщиков на 14 дней; приоритет — RuStore.
  На горизонте 2027 — глобальная верификация Android-разработчиков
  Google даже для sideload (следить за новостями).
