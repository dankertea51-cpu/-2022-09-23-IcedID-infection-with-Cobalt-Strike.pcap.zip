# Индикаторы компрометации

Инцидент: `2022-09-23-IcedID-infection-with-Cobalt-Strike` · Аналитик: DeqUser · Извлечено: **2026-03-26** · TLP:CLEAR

| Файл | Формат | Для чего |
|---|---|---|
| [`iocs.csv`](iocs.csv) | CSV | импорт в таблицы, TIP, скрипты обогащения |
| [`iocs.json`](iocs.json) | JSON | автоматическая обработка, загрузка в MISP/SIEM |
| [`blocklist.txt`](blocklist.txt) | plain text | межсетевые экраны, DNS-синкхолы, denylist прокси |

## Сводка

| Тип | Значение | Вердикт |
|---|---|---|
| domain | `trallfasterinf.com` | 🔴 вредоносный (10/92 VT) |
| domain | `win-cosmic-mind.monasticservice.org` | 🟠 подозрительный (SMB-аномалия) |
| domain | `ctldl.windowsupdate.com` | 🟡 легитимный, требует проверки частоты |
| ipv4 | `137.184.114.20` | 🔴 вредоносный (DigitalOcean) |
| ipv4 | `209.197.3.8` | 🟢 легитимный (Microsoft) |
| ipv4 | `10.9.23.23` | 🟠 внутренний хост под подозрением |
| port | `445/TCP` исходящий | 🔴 критический индикатор |

## Как применять

```bash
# Только то, что подлежит блокировке (без комментариев и MONITOR-строк)
grep -v '^#' ioc/blocklist.txt | grep -v '^$'

# Выборка вредоносных индикаторов из CSV
awk -F, '$6=="malicious"{print $2}' ioc/iocs.csv

# Разбор JSON
jq -r '.indicators[] | select(.verdict=="malicious") | .value' ioc/iocs.json
```

> ⚠️ `209.197.3.8` и `ctldl.windowsupdate.com` блокировать **не следует** — это инфраструктура Microsoft. `10.9.23.23` — внутренний хост: его нужно изолировать и проверить, а не блокировать на периметре.
