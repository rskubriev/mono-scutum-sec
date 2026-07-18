# Scutum Security Workflow

[English version](README.md)

Переиспользуемый workflow GitHub Actions для безопасной сборки Docker-образов. До публикации он запускает Hadolint и Trivy, создаёт CycloneDX SBOM через Syft, а после публикации может прикрепить SBOM и подписать образ Cosign в keyless-режиме.

## Подключение

Создайте в репозитории с Dockerfile файл `.github/workflows/container-security.yml`:

```yaml
name: Container security

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write

jobs:
  security:
    uses: rskubriev/mono-scutum-sec/.github/workflows/docker-security.yml@main
    with:
      image-name: ghcr.io/OWNER/IMAGE
      push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      sign: true
```

Замените `OWNER/IMAGE` на путь к вашему образу в registry. Для pull request workflow проверит исходники и образ, но не будет его публиковать. При push в `main` он публикует образ с тегом SHA коммита, сохраняет SBOM как артефакт, прикрепляет его к образу и подписывает через GitHub OIDC.

## Параметры

| Параметр | По умолчанию | Назначение |
| --- | --- | --- |
| `context` | `.` | Контекст сборки Docker в вызывающем репозитории. |
| `dockerfile` | `Dockerfile` | Путь к Dockerfile относительно корня вызывающего репозитория. |
| `image-name` | пусто | Полное имя образа, например `ghcr.io/owner/image`. Обязательно при `push: true`. |
| `push` | `false` | Публиковать образ после успешных проверок. |
| `sign` | `true` | Подписывать опубликованный образ Cosign без хранения ключей. |

## Версии

На этапе активной разработки используйте `@main`, чтобы получать актуальную версию workflow. Для стабильных выпусков будут применяться мажорные теги, например `@v1`; такой тег обновляется только обратно совместимыми изменениями. Для максимального контроля цепочки поставки закрепляйте workflow на полном SHA коммита и обновляйте его осознанно.

## Модель безопасности

Все сторонние контейнерные инструменты и GitHub Actions закреплены неизменяемыми digest или commit SHA. По умолчанию workflow получает права только на чтение; права на публикацию пакетов и OIDC-подпись объявлены только для job сборки. При `push: true` вызывающему репозиторию нужно разрешить GitHub Actions записывать пакеты.

Готовый пример находится в `examples/caller-workflow.yml`. В этом репозитории намеренно нет кода приложений, Dockerfile, учётных данных или тестовых образов.
