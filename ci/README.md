# CI-конфигурации (готовые к установке)

Эти два workflow лежат отдельно от `.github/workflows/`, потому что автоматизация, создавшая коммит, не имеет прав на публикацию workflow-файлов в GitHub. Чтобы включить их, перенесите файлы вручную:

```bash
mkdir -p .github/workflows
git mv ci/validate.yml .github/workflows/validate.yml
git mv ci/pages.yml    .github/workflows/pages.yml
git commit -m "Enable CI workflows" && git push
```

| Файл | Что делает |
|---|---|
| [`validate.yml`](validate.yml) | Проверяет синтаксис `ioc/iocs.json`, `ioc/iocs.csv`, правил Sigma и непустоту блок-листа при каждом push/PR |
| [`pages.yml`](pages.yml) | Публикует каталог `docs/` на GitHub Pages (веб-версия отчёта) |

Для `pages.yml` дополнительно включите Pages в настройках репозитория: **Settings → Pages → Source: GitHub Actions**.
