# Методология анализа

Документ описывает воспроизводимый порядок разбора PCAP-дампа и конкретные фильтры, которыми получены выводы из [отчёта](report.md).

[← Вернуться к README](../README.md)

---

## Этап 1. Общий обзор дампа

Цель — понять масштаб записи, состав протоколов и участников обмена.

| Wireshark | Назначение |
|---|---|
| `Statistics → Capture File Properties` | длительность записи, число пакетов |
| `Statistics → Protocol Hierarchy` | доля протоколов, аномальные доли SMB/DNS |
| `Statistics → Conversations` | топ-собеседники по объёму и числу потоков |
| `Statistics → Endpoints` | список внешних IP для последующей проверки |

Аналог в CLI:

```bash
capinfos capture.pcap
tshark -r capture.pcap -q -z io,phs
tshark -r capture.pcap -q -z conv,tcp
```

---

## Этап 2. Анализ DNS

Злоумышленники почти всегда резолвят свои серверы, поэтому DNS — самый быстрый путь к списку кандидатов на C2.

```bash
# Все уникальные запрошенные имена
tshark -r capture.pcap -Y 'dns.flags.response == 0' \
  -T fields -e dns.qry.name | sort -u

# Имена, которые не разрезолвились (NXDOMAIN) — признак мёртвых C2
tshark -r capture.pcap -Y 'dns.flags.rcode == 3' \
  -T fields -e dns.qry.name | sort -u
```

Фильтры Wireshark:

```
dns
dns.qry.name contains "trallfasterinf"
dns && !(dns.qry.name matches "(microsoft|windows|akamai)\\.com$")
```

---

## Этап 3. HTTP / HTTPS

```bash
# Все HTTP-запросы: хост, метод, URI, User-Agent
tshark -r capture.pcap -Y http.request \
  -T fields -e ip.dst -e http.host -e http.request.method \
  -e http.request.uri -e http.user_agent

# Извлечение переданных объектов (EXE, DLL, документы)
tshark -r capture.pcap --export-objects http,./exported/
```

Фильтры Wireshark:

```
http.request
http.response.code == 200 && http.content_type contains "application"
tls.handshake.type == 1            # Client Hello, поле SNI
tls.handshake.extensions_server_name
```

---

## Этап 4. Поиск аномалий портов и SMB наружу

Ключевая находка этого разбора получена именно здесь.

```bash
# Исходящий SMB на адреса вне внутренних диапазонов
tshark -r capture.pcap -Y 'tcp.dstport == 445 && !(ip.dst == 10.0.0.0/8) \
  && !(ip.dst == 192.168.0.0/16) && !(ip.dst == 172.16.0.0/12)' \
  -T fields -e ip.src -e ip.dst

# Нестандартные порты назначения
tshark -r capture.pcap -Y 'tcp.flags.syn == 1 && tcp.flags.ack == 0' \
  -T fields -e ip.dst -e tcp.dstport | sort | uniq -c | sort -rn
```

Что считается аномалией:

- SMB (445/TCP, 139/TCP) в сторону интернета;
- «сырой» HTTP на нестандартных портах (8080, 8443, 4444);
- регулярные соединения с ровным интервалом — признак beacon-активности C2 (Cobalt Strike);
- обращения к IP-адресам без предшествующего DNS-запроса.

---

## Этап 5. Обогащение и проверка репутации

Каждый найденный домен/IP проверяется по внешним источникам:

| Источник | Что даёт |
|---|---|
| [VirusTotal](https://www.virustotal.com/) | агрегированные вердикты AV-движков |
| WHOIS / RDAP | владелец, дата регистрации (свежий домен = красный флаг) |
| ASN lookup | хостинг-провайдер (DigitalOcean, Hetzner и т.п. часто используются под фишинг) |
| Passive DNS | другие домены на том же IP |

> Отсутствие детектов **не является** доказательством безопасности: свежие C2-домены попадают в базы с задержкой в дни и недели.

---

## Этап 6. Оформление результатов

1. Составить список IoC (домены, IP, порты, хеши) — см. [`ioc/`](../ioc/).
2. Написать правила детекта — см. [`rules/`](../rules/).
3. Сформулировать рекомендации по блокировке и мониторингу.
4. Указать классификацию распространения (TLP) и дату актуальности индикаторов.

---

## Ограничения

- Дамп разбирался без доступа к телеметрии хостов, поэтому процессы и файлы, породившие трафик, не подтверждены.
- Точные метрики объёма записи не фиксировались, выводы о масштабе атаки не делаются.
- Расшифровка TLS не выполнялась: анализ шифрованных сессий ограничен метаданными (SNI, JA3, размеры и тайминги пакетов).
