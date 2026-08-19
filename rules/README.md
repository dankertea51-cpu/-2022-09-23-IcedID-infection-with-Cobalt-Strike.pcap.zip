# Правила детекта

Готовые к развёртыванию сигнатуры по индикаторам из [отчёта](../docs/report.md).

| Файл | Платформа | Что покрывает |
|---|---|---|
| [`suricata.rules`](suricata.rules) | Suricata / Snort | DNS и HTTP к фишинговому домену, соединения с `137.184.114.20`, исходящий SMB наружу |
| [`sigma/icedid_phishing_domain.yml`](sigma/icedid_phishing_domain.yml) | Sigma → любой SIEM | Резолв домена `trallfasterinf.com` и коннекты к фишинговому IP |
| [`sigma/outbound_smb_to_internet.yml`](sigma/outbound_smb_to_internet.yml) | Sigma → любой SIEM | SMB (445/TCP) за пределы периметра + резолв SMB-аномального домена |

## Развёртывание

**Suricata**

```bash
sudo cp suricata.rules /etc/suricata/rules/icedid-2022-09-23.rules
# добавьте файл в rule-files в suricata.yaml, затем:
sudo suricata -T -c /etc/suricata/suricata.yaml   # тест конфигурации
sudo systemctl reload suricata
```

**Sigma → конвертация под свой SIEM**

```bash
pip install sigma-cli
sigma convert -t splunk        sigma/            # Splunk SPL
sigma convert -t microsoft365defender sigma/     # KQL
sigma convert -t elasticsearch -f siem_rule sigma/
```

## Замечания

- SID из локального диапазона `9000001-9000007` — при необходимости переназначьте под свою нумерацию.
- Правило на исходящий SMB носит характер **policy-violation**: перед включением в блокирующем режиме проверьте легитимные облачные файловые сервисы (например, Azure Files) и внесите их в исключения.
- Правило-охотник по `ctldl.windowsupdate.com` закомментировано намеренно — домен легитимный, порог срабатывания нужно калибровать под свой трафик.
