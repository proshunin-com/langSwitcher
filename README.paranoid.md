# langSwitcher — paranoid fork

Личный форк [reg2005/langSwitcher](https://github.com/reg2005/langSwitcher) с физически выпиленным SQLite-логированием.

Это ветка `paranoid`. Основная ветка `main` остаётся зеркалом upstream для удобного обновления — при необходимости новые коммиты автора cherry-pick'аются в `paranoid`.

## Зачем форк

Сам upstream-проект чистый: исходники открыты, App Sandbox активен, сеть запрещена ядром, логирование в SQLite по умолчанию выключено. Аудит подтвердил отсутствие сетевой активности и в коде, и в runtime.

Тем не менее: код для логирования физически присутствует в бинарнике. Теоретически его можно случайно или намеренно (через настройку, баг или вредоносный коммит апстрима) включить. Defense-in-depth подход — **удалить** этот код полностью, а не полагаться только на toggle в настройках.

После форка:

- В бинарнике **нет кода для записи на диск** (кроме UserDefaults для настроек самой программы)
- Папка `~/Library/Application Support/LangSwitcher/` не создаётся при запуске
- Entitlement `files.user-selected.read-write` убран (был нужен только для JSON-экспорта логов)

## Что выпилено

| Файл | Действие |
|---|---|
| `Sources/Services/ConversionLogStore.swift` | удалён целиком |
| `Sources/Models/ConversionLog.swift` | удалён целиком |
| `Sources/Views/ConversionLogView.swift` | удалён целиком |
| `Sources/Views/SettingsView.swift` | убрана 5-я вкладка Log + tabItems entry |
| `Sources/App/AppDelegate.swift` | убран `conversionLogStore`, 3 вызова `logConversion()`, сам метод |
| `Sources/Services/SettingsManager.swift` | убраны `loggingEnabled`, `logMaxEntries` (свойства, ключи, init) |
| `Sources/Localization/Strings_*.swift` | удалены 19+19 локализационных ключей `log.*` и `settings.tab.log` |
| `LangSwitcher.entitlements` | убран `files.user-selected.read-write` |
| `LangSwitcher.xcodeproj/project.pbxproj` | очищены ссылки на удалённые файлы |

Вся работа выполнена одним коммитом в ветке `paranoid`. Тесты (93 штуки) проходят полностью.

## Сборка

Требования: macOS 13+, Xcode 16+ (тестировалось на Xcode 26.4.1).

```bash
git clone -b paranoid git@github.com:proshunin-com/langSwitcher.git
cd langSwitcher

# тесты
xcodebuild test \
  -project LangSwitcher.xcodeproj \
  -scheme LangSwitcher \
  -destination 'platform=macOS' \
  CODE_SIGN_IDENTITY="-" \
  CODE_SIGNING_REQUIRED=NO \
  CODE_SIGNING_ALLOWED=NO

# release-сборка
xcodebuild -project LangSwitcher.xcodeproj \
  -scheme LangSwitcher \
  -configuration Release \
  -derivedDataPath build \
  -arch arm64 \
  ONLY_ACTIVE_ARCH=NO \
  CODE_SIGN_IDENTITY="-" \
  CODE_SIGNING_REQUIRED=NO \
  CODE_SIGNING_ALLOWED=NO \
  build

# ad-hoc подпись и установка
codesign --force --deep --sign - build/Build/Products/Release/LangSwitcher.app
cp -R build/Build/Products/Release/LangSwitcher.app /Applications/
open /Applications/LangSwitcher.app
```

## Runtime-проверки после установки

```bash
# SQLite-папка не должна создаваться никогда
ls ~/Library/Application\ Support/LangSwitcher 2>&1
# ожидаем: No such file or directory

# Нет сетевых сокетов
PID=$(pgrep -f "/Applications/LangSwitcher.app" | head -1)
lsof -p $PID 2>/dev/null | awk '{print $5}' | sort -u
# ожидаем: только REG, DIR, CHR, unix, KQUEUE — никаких IPv4/IPv6
```

## Обновление от upstream

Когда у автора выходят новые версии:

```bash
git fetch upstream
git checkout main
git merge upstream/main
git push origin main

# теперь применяем изменения в paranoid
git checkout paranoid
git rebase main
# при конфликтах — пересмотреть нужно ли тащить новые ConversionLog* фичи (нет, не нужно)
git push origin paranoid --force-with-lease
```

## Лицензия

MIT — как у upstream. См. `LICENSE`.

## Автор форка

Max Proshunin / NSK / DeFacto — для личного использования.
Авторство upstream: [reg2005](https://github.com/reg2005).
