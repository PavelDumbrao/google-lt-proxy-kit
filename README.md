# Сервисы Google и Claude Code через литовский прокси

Набор для ИИ-агента ученика: он сам ставит Clash Verge Rev и настраивает компьютер так, что
**Gemini, NotebookLM, AI Studio, Google Antigravity** и вход в аккаунт Google идут через литовский прокси.
Через тот же прокси агент пускает **Claude Code** (и с ним Claude Desktop и claude.ai).
Весь остальной интернет идёт как обычно.

Работает на macOS и Windows.

## Как пользоваться

1. Возьмите у преподавателя файл прокси (`proxy_credentials.txt`) и сохраните его на компьютер.
2. Откройте своего агента (Claude Code, Codex, ZCode) и вставьте:

```text
Открой https://github.com/PavelDumbrao/google-lt-proxy-kit и выполни задание из файла AGENT.md
(сырой текст: https://raw.githubusercontent.com/PavelDumbrao/google-lt-proxy-kit/main/AGENT.md).
Мой файл прокси: <путь к proxy_credentials.txt>
```

3. Делайте то, что попросит агент: пару кликов в Clash Verge Rev и пароль от компьютера при установке служебного режима.
4. В конце полностью закройте и снова откройте Claude Code — он пойдёт через прокси.

Если Claude Code у вас пока не запускается, дайте это задание другому агенту (Codex, ZCode):
он настроит прокси, и Claude Code заработает.

## Что внутри

- [`AGENT.md`](AGENT.md) — пошаговое задание для агента.
- [`config/google-lt.template.yaml`](config/google-lt.template.yaml) — шаблон профиля: Google и Claude → прокси, остальное напрямую.

## Как выключить

Clash Verge Rev → выключить «Режим TUN» или закрыть приложение. Интернет вернётся как был.

## Безопасность

- Файл прокси и готовый профиль — личные. Не публикуйте и не пересылайте.
- В репозитории нет ни одного логина и пароля: агент подставляет их у вас на компьютере.
