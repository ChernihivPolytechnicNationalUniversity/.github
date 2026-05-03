<img
  align="left"
  width="210"
  hspace="28"
  src="https://raw.githubusercontent.com/devicons/devicon/master/icons/github/github-original-wordmark.svg"
  alt="Repository illustration"
/>

Всі репозиторії організації дотримуються єдиної структури. Це спрощує
навігацію, онбординг нових учасників та уніфікує CI/CD.

<br clear="left" />

## Структура

```text
repo/
├── .github/
│   └── workflows/
│       ├── ci-build.yml          # CI: перевірка збірки на кожен PR
│       └── *-deploy.yml          # CD: build + deploy на push в dev та tag v*
├── .infra/
│   └── helm/
│       ├── Chart.yaml
│       ├── templates/
│       ├── values.yaml           # base values
│       ├── values-dev.yaml       # dev overrides
│       └── values-prod.yaml      # prod overrides
├── <app>/                        # код застосунку (ui/, back/, тощо)
│   ├── Dockerfile
│   ├── src/
│   └── ...
├── docs/                         # документація проєкту
├── .gitignore
├── .env.example                  # приклад змінних оточення
└── README.md
```

## GitHub workflows

Містить CI/CD workflows.

**ci-build.yml** запускається на кожен PR у `dev` та `main`. Перевіряє що
проєкт збирається. Використовується як required status check в
[Require CI](./branch-strategy.md#rulesets) ruleset.

**Deploy workflow** (назва залежить від репо) запускається на push в `dev` та
на push тегу `v*.*.*`. Збирає Docker-образ, пушить в GitHub Container Registry
та деплоїть через Helm.

## Infra

Інфраструктурна конфігурація для Kubernetes. Кожен репозиторій містить свій
Helm chart в `.infra/helm/`.

**Chart.yaml** описує метадані chart-у: назва, опис, версія.

**templates/** містить Kubernetes-маніфести: deployment, service, ingress.
Шаблони параметризовані через values і однакові для dev та prod – відрізняється
тільки конфігурація.

**values.yaml** – базові значення, спільні для обох оточень: image pull policy,
ресурси (CPU, memory), liveness/readiness probes, стратегія деплою
(RollingUpdate).

**values-dev.yaml** – overrides для dev: хост ingress, кількість реплік (як
правило 1), environment-specific змінні (logging level, debug flags, swagger).

**values-prod.yaml** – overrides для prod: production хост, більше реплік (як
правило 2), строгіші налаштування (swagger вимкнений, logging level INFO).

Deploy workflow під час деплою виконує `helm upgrade --install` з потрібним
values-файлом та передає `image.tag` з версією зібраного Docker-образу.

## App

Код застосунку в окремій директорії. Назва залежить від проєкту: `ui/` для
фронтенду, `back/` для бекенду. Всередині – Dockerfile, вихідний код та
конфігурація збірки.

Dockerfile є точкою входу для CI/CD – і ci-build.yml, і deploy workflow
збирають образ саме з нього.

## Docs

Проєктна документація. Не організаційна – організаційні стандарти живуть в
[.github](https://github.com/ChernihivPolytechnicNationalUniversity/.github)
репозиторії.

## Кореневі файли

- **README.md** – опис проєкту, tech stack, інструкції з запуску
- **.gitignore** – стандартний для стеку технологій
- **.env.example** – приклад змінних оточення для локальної розробки. `.env`
  файл не комітиться
