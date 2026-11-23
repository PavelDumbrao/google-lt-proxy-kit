# Задание для ИИ-агента: Claude Code через прокси ученика — всё сам, с нуля

Ты — агент ученика (OpenCode, Codex, ZCode или похожий). Сделай так, чтобы **Claude Code в терминале
работал через прокси ученика**. Всё ставишь и настраиваешь сам, с нуля: ученик ничего не кликает и не
вводит пароль от компьютера. Его участие — только вход в свой аккаунт Claude в конце (шаг 7).

Права администратора и `sudo` не нужны. Если какая-то команда их просит — не проси пароль у ученика,
найди способ без них.

Объясняй простыми словами, что делаешь. Не останавливайся на полпути: делай шаги по порядку до отчёта.
Спрашивай ученика только там, где написано «спроси».

## Как это устроено

- Ты ставишь маленькую программу **mihomo** в папку ученика. Она слушает только этот компьютер:
  `127.0.0.1:17890`.
- mihomo отправляет запросы к Claude и Google через прокси ученика, всё остальное — напрямую.
- Claude Code получает адрес `http://127.0.0.1:17890` в своих настройках. Сам Claude Code SOCKS-прокси
  не поддерживает (https://code.claude.com/docs/en/network-config), поэтому и нужен mihomo.
- Системные настройки сети, браузер и остальной интернет ученика не меняются.

## Что дал ученик

Строку прокси `host:port:login:password` (протокол SOCKS5) — прямо в сообщении или путём к файлу.
Строки с `#` — комментарии. Если строки нет — спроси.

## Правила безопасности (обязательно)

- **Не повторяй логин и пароль прокси** в ответах, отчёте и логах. Показывай только `host:port`.
- Файл с паролем (`config.yaml`) лежит только в папке ученика, права `600` на macOS / Linux.
  Не клади его в git и не отправляй в сторонние сервисы.
- Скачивай программы только с официальных адресов из этого файла.
- Если уже стоит Clash / Clash Verge / другой VPN — ничего в нём не меняй и не выключай. Он не мешает:
  mihomo работает на своём порту.
- Пароли от аккаунтов ученик вводит сам.

## Шаг 0. Осмотреться

1. Определи систему и процессор: macOS (Apple Silicon `arm64` или Intel `x86_64`), Windows (`AMD64` или
   `ARM64`), Linux.
2. Разбери строку прокси на HOST, PORT, LOGIN, PASSWORD.
3. Если папка `~/.mihomo-claude` уже есть — это повторный запуск: обнови файлы и перезапусти службу,
   а не ставь второй раз.
4. Проверь, свободен ли порт `17890`. Если занят не твоим mihomo — возьми `17891` и подставляй его везде
   ниже.
5. Проверь `claude --version` и прочитай блок `env` в `~/.claude/settings.json`
   (Windows: `%USERPROFILE%\.claude\settings.json`), если файл есть.
   Если там задан `ANTHROPIC_BASE_URL` — ученик ходит в Claude через шлюз: **спроси**, оставить шлюз или
   перейти на прокси. Без его «да» эту строку не трогай.

## Шаг 1. Проверить прокси

macOS / Linux:
```bash
curl -s -m 25 --proxy "socks5h://HOST:PORT" --proxy-user "LOGIN:PASSWORD" https://ipinfo.io/json
curl -s -m 25 --proxy "socks5h://HOST:PORT" --proxy-user "LOGIN:PASSWORD" https://api.anthropic.com/cdn-cgi/trace
```
Windows: `curl.exe` с теми же аргументами.

Ожидается: страна прокси (например `"country": "LT"`) и строка `loc=` с той же страной.
- Ошибка входа (`authentication failed`, `rejected`) — неверные логин или пароль: остановись, скажи ученику
  взять свежую строку прокси.
- Таймаут — проверь интернет без прокси. Интернет есть, а прокси молчит — остановись и сообщи.

## Шаг 2. Скачать mihomo

Официальные релизы: https://github.com/MetaCubeX/mihomo/releases

1. Узнай последнюю версию (без API и лимитов):
   ```bash
   curl -sI -o /dev/null -w "%{redirect_url}\n" https://github.com/MetaCubeX/mihomo/releases/latest
   ```
   Ответ заканчивается на `/tag/vX.Y.Z` — это версия `TAG`.
2. Скачай файл `https://github.com/MetaCubeX/mihomo/releases/download/TAG/ФАЙЛ`:

   | Система | ФАЙЛ |
   | - | - |
   | macOS Apple Silicon | `mihomo-darwin-arm64-TAG.gz` |
   | macOS Intel | `mihomo-darwin-amd64-compatible-TAG.gz` |
   | Windows x64 | `mihomo-windows-amd64-compatible-TAG.zip` |
   | Windows ARM | `mihomo-windows-arm64-TAG.zip` |
   | Linux x64 | `mihomo-linux-amd64-compatible-TAG.gz` |

3. Распакуй в папку `~/.mihomo-claude` (Windows: `%USERPROFILE%\.mihomo-claude`) под именем `mihomo`
   (Windows: `mihomo.exe`). macOS / Linux: `gunzip`, затем `chmod +x`.
4. Проверь: `~/.mihomo-claude/mihomo -v` печатает версию.

Если на macOS уже есть `mihomo` из Homebrew — можно взять его и не скачивать.

## Шаг 3. Сделать конфиг

1. Скачай шаблон:
   https://raw.githubusercontent.com/PavelDumbrao/google-lt-proxy-kit/main/config/claude-code.template.yaml
2. Замени `HOST`, `PORT`, `LOGIN`, `PASSWORD` в блоке `proxies`. Логин и пароль возьми в двойные кавычки.
   Порт в `mixed-port` меняй, только если `17890` занят. Больше ничего не трогай.
3. Сохрани как `~/.mihomo-claude/config.yaml`. macOS / Linux: `chmod 600`.
4. Проверь: `~/.mihomo-claude/mihomo -t -d ~/.mihomo-claude` → `test is successful`.

## Шаг 4. Запустить mihomo и поставить в автозапуск

Цель: mihomo запущен сейчас и сам стартует после перезагрузки, без окна и без прав администратора.

**macOS.** Создай `~/Library/LaunchAgents/local.mihomo-claude.plist` (пути — полные, без `~`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>local.mihomo-claude</string>
  <key>ProgramArguments</key><array>
    <string>/Users/ИМЯ/.mihomo-claude/mihomo</string>
    <string>-d</string>
    <string>/Users/ИМЯ/.mihomo-claude</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/Users/ИМЯ/.mihomo-claude/mihomo.log</string>
  <key>StandardErrorPath</key><string>/Users/ИМЯ/.mihomo-claude/mihomo.log</string>
</dict></plist>
```
Запусти (при повторной установке сначала выгрузи старую службу):
```bash
launchctl bootout "gui/$(id -u)/local.mihomo-claude" 2>/dev/null
launchctl bootstrap "gui/$(id -u)" ~/Library/LaunchAgents/local.mihomo-claude.plist
```
macOS может показать уведомление «Добавлен фоновый объект» — это нормально, скажи об этом ученику.

**Windows (PowerShell).** Запусти без окна и добавь в автозапуск текущего пользователя:
```powershell
$d = "$env:USERPROFILE\.mihomo-claude"
Start-Process -FilePath "$d\mihomo.exe" -ArgumentList "-d", "`"$d`"" -WindowStyle Hidden
$cmd = "powershell.exe -NoProfile -WindowStyle Hidden -Command `"Start-Process -FilePath '$d\mihomo.exe' -ArgumentList '-d','$d' -WindowStyle Hidden`""
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "MihomoClaude" -Value $cmd
```
Если Windows спросит про доступ к сети для mihomo — он слушает только этот компьютер, можно отказать.
Если этот способ не сработал — добейся той же цели другим способом без прав администратора
(например, задача в Планировщике «при входе в систему» для текущего пользователя).

**Linux.** Служба `systemd --user` с командой `~/.mihomo-claude/mihomo -d ~/.mihomo-claude`, `enable --now`.

## Шаг 5. Проверить локальный порт

```bash
curl -s -m 25 --proxy http://127.0.0.1:17890 https://api.anthropic.com/cdn-cgi/trace
curl -s -m 25 -o /dev/null -w "%{http_code}\n" --proxy http://127.0.0.1:17890 https://api.anthropic.com/v1/messages
curl -s -m 25 -o /dev/null -w "%{http_code}\n" --proxy http://127.0.0.1:17890 https://github.com/
```
Ожидается: `loc=` со страной прокси, затем `405`, затем `200`.

Не получилось — смотри `~/.mihomo-claude/mihomo.log`, исправь и повтори шаги 3–5. Если mihomo поднять
так и не удалось — переходи к разделу «Запасной путь» в конце.

## Шаг 6. Claude Code: установить и подключить к прокси

1. Если `claude --version` не работает — установи Claude Code официальным установщиком **через
   локальный порт** (напрямую установщик может не скачаться из-за региона).

   macOS / Linux:
   ```bash
   export HTTPS_PROXY=http://127.0.0.1:17890 HTTP_PROXY=http://127.0.0.1:17890
   curl -fsSL https://claude.ai/install.sh | bash
   ```
   Windows (CMD — этот установщик качает через `curl`, а он слушается переменной):
   ```batch
   set HTTPS_PROXY=http://127.0.0.1:17890
   curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
   ```
   На Windows желательно поставить Git for Windows (https://git-scm.com/downloads/win), если его нет.
   Проверь в новом терминале: `claude --version`. Если команда не найдена — добавь папку установки в PATH
   (её называет установщик; на macOS / Linux обычно `~/.local/bin`).

2. Пропиши прокси в настройках Claude Code. Файл `~/.claude/settings.json`
   (Windows: `%USERPROFILE%\.claude\settings.json`). Если файл есть — сначала сделай копию
   `settings.json.bak` и **добавь** три строки в блок `env`, остальное не трогай. Если файла нет — создай.
   Файл должен остаться корректным JSON.
   ```json
   {
     "env": {
       "HTTPS_PROXY": "http://127.0.0.1:17890",
       "HTTP_PROXY": "http://127.0.0.1:17890",
       "NO_PROXY": "localhost,127.0.0.1"
     }
   }
   ```
   Если там уже был другой `HTTPS_PROXY` — **спроси** ученика перед заменой.

## Шаг 7. Вход в аккаунт Claude (единственное, что делает ученик)

Сначала проверь, не выполнен ли вход: `claude -p "Ответь одним словом: работает"`. Пришёл ответ — вход
есть, переходи к шагу 8.

Если нужен вход: обычный браузер ученика идёт мимо прокси, и страница Claude в нём может не открыться.
Открой отдельное окно браузера, которое идёт через mihomo. Подойдёт любой браузер на Chromium: Chrome,
Edge, Яндекс Браузер, Brave.

macOS:
```bash
open -na "Google Chrome" --args --proxy-server="http://127.0.0.1:17890" --user-data-dir="$HOME/.mihomo-claude/browser" https://claude.ai/login
```
Windows (PowerShell; Edge есть на любом Windows):
```powershell
Start-Process msedge.exe -ArgumentList "--proxy-server=http://127.0.0.1:17890", "--user-data-dir=`"$env:USERPROFILE\.mihomo-claude\browser`"", "https://claude.ai/login"
```

Дальше веди ученика по шагам и жди его ответа:
1. В открывшемся окне войти в свой аккаунт Claude.
2. В отдельном окне терминала выполнить `claude`, выбрать вход через аккаунт Claude.
3. Claude Code покажет длинную ссылку. Скопировать её и открыть **в том же окне браузера**, где уже
   выполнен вход (не в обычном браузере), и подтвердить доступ.
4. Если Claude Code попросит код — скопировать его со страницы и вставить в терминал.

Если на macOS нет ни одного браузера на Chromium — **спроси** ученика, можно ли поставить Google Chrome
(https://www.google.com/chrome/), и поставь.

## Шаг 8. Итоговая проверка

```bash
claude -p "Ответь одним словом: работает"
```
Пришёл ответ — готово. Дополнительно: в `~/.mihomo-claude/mihomo.log` нет ошибок соединения с прокси.

Если ответ — ошибка про недоступный регион или `403`, а шаг 5 зелёный: проверь, что три строки в
`settings.json` на месте и терминал открыт заново. Если всё так — дело в самом аккаунте Claude,
прокси это не исправит; скажи об этом ученику.

## Шаг 9. Отчёт ученику

Коротко, без логина и пароля:
- что установлено и где: mihomo (версия) в `~/.mihomo-claude`, Claude Code (версия);
- адрес локального порта и `host:port` прокси;
- результаты шагов 1, 5 и 8;
- как пользоваться: просто запускать `claude` в терминале; mihomo стартует сам после перезагрузки;
- как открыть claude.ai в браузере через прокси: команда из шага 7;
- как выключить: macOS — `launchctl bootout "gui/$(id -u)/local.mihomo-claude"`; Windows — завершить
  процесс `mihomo.exe`. Пока mihomo выключен, Claude Code не работает;
- как удалить полностью: остановить mihomo, убрать автозапуск (plist на macOS, значение `MihomoClaude` в
  реестре на Windows), удалить папку `~/.mihomo-claude` и три строки прокси из `settings.json`.

## Запасной путь: без mihomo

Только если mihomo не удалось скачать или запустить. Многие прокси принимают и протокол HTTP на том же
порту. Проверь:
```bash
curl -s -m 25 --proxy "http://HOST:PORT" --proxy-user "LOGIN:PASSWORD" https://api.anthropic.com/cdn-cgi/trace
```
Если пришёл `loc=` со страной прокси — пропиши в `~/.claude/settings.json` в блок `env` строки
`HTTPS_PROXY` и `HTTP_PROXY` со значением `http://LOGIN:PASSWORD@HOST:PORT` (спецсимволы в логине и
пароле закодируй по правилам URL) и `NO_PROXY` как в шаге 6. macOS / Linux: `chmod 600` на файл.
Скажи ученику, что в этом варианте пароль прокси лежит в настройках Claude Code и через прокси идёт весь
трафик Claude Code. Затем шаги 7–9; окно браузера для входа в этом варианте открыть не получится —
ученику нужен свой способ открыть claude.ai.

## Важно знать ученику

- Через прокси идут только Claude и Google из Claude Code и из отдельного окна браузера. Остальной интернет
  не меняется.
- Строку прокси никому не пересылайте.
- Приложение Claude Desktop этим способом не настраивается — только Claude Code в терминале.
