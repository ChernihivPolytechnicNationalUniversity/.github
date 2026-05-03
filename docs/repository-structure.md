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

## Типи репозиторіїв

Репозиторії діляться на два типи за кількістю оточень.

**Multi-env** – має окремі dev та prod оточення. Деплой на dev відбувається на
push в `dev`, на prod – на push тегу `v*`. У `.infra/helm/` лежать
`values-dev.yaml` та `values-prod.yaml` поверх `values.yaml`.

**Mono-env** – має тільки одне production-оточення. Деплой відбувається тільки
на push тегу `v*`. У `.infra/helm/` лежить лише `values.yaml`.

## Структура

```text
repo/
├── .github/
│   └── workflows/
│       ├── ci-build.yml          # CI: перевірка збірки на кожен PR
│       └── deploy.yml            # CD: build + deploy
├── .infra/
│   └── helm/
│       ├── Chart.yaml
│       ├── templates/
│       ├── values.yaml           # base values
│       ├── values-dev.yaml       # multi-env: dev overrides
│       └── values-prod.yaml      # multi-env: prod overrides
├── <app>/                        # код застосунку (ui/, back/, тощо)
│   ├── Dockerfile
│   ├── docker-compose.yml        # для локальної розробки
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

**deploy.yml** збирає Docker-образ, пушить в GitHub Container Registry та
деплоїть через Helm. Тригери залежать від типу репозиторію:

- Multi-env:
  - push в `dev` → dev-оточення
  - реліз через `dev → main` + push тегу `v*` на `main` → prod-оточення
- Mono-env: реліз через `dev → main` + push тегу `v*` на `main` → production

Деталі релізного процесу описані в [release-process](./release-process.md).

## Shared actions

Стандарти CI/CD реалізовані через
[shared-actions](https://github.com/ChernihivPolytechnicNationalUniversity/shared-actions)
– набір reusable composite GitHub Actions, які використовують усі репозиторії
організації. Замість того, щоб кожен репо мав свою копію логіки збірки,
деплою та нотифікацій, всі вони викликають одні й ті самі actions.

Доступні actions:

- **docker-build-push** – збирає Docker-образ та пушить у GHCR з semver
  тегуванням і registry cache
- **k8s-azure-login** – Azure federated OIDC login, kubelogin та підготовка
  kubeconfig для деплою через Helm
- **collect-commits** – формує base64-encoded HTML список комітів між поточним
  та попереднім semver тегом для нотифікації про реліз
- **notify** – відправляє CI або deploy нотифікацію через
  [notification-service](https://github.com/ChernihivPolytechnicNationalUniversity/notification-service)

Це дозволяє підтримувати єдиний стандарт CI/CD: коли змінюється логіка
деплою, оновлення доходить до всіх репо одночасно через `@main` reference у
shared-actions.

## GitHub Environments

На кожному репозиторії мають бути налаштовані два environments:

- **production**
- **development**

Це обов'язкова вимога: GitHub Environments використовуються для авторизації в
Entra ID під час деплою. Без них workflow не зможе отримати токен для пушу
секретів та інтеграцій.

Налаштовуються в **Settings → Environments**.

## Infra

Інфраструктурна конфігурація для Kubernetes. Кожен репозиторій містить свій
Helm chart в `.infra/helm/`.

**Chart.yaml** описує метадані chart-у: назва, опис, версія.

**templates/** містить Kubernetes-маніфести: deployment, service, ingress.
Шаблони параметризовані через values – відрізняється тільки конфігурація між
оточеннями.

**values.yaml** – базові значення для multi-env (спільні для dev та prod) або
єдиний values-файл для mono-env. Містить image pull policy, ресурси (CPU,
memory), liveness/readiness probes, стратегію деплою.

**values-dev.yaml** (тільки multi-env) – overrides для dev: хост ingress,
кількість реплік (як правило 1), environment-specific змінні (logging level,
debug flags, swagger).

**values-prod.yaml** (тільки multi-env) – overrides для prod: production хост,
більше реплік (як правило 2), строгіші налаштування.

deploy.yml виконує `helm upgrade --install` з потрібним values-файлом та
передає `image.tag` з версією зібраного Docker-образу.

## App

Код застосунку в окремій директорії. Назва залежить від проєкту: `ui/` для
фронтенду, `back/` для бекенду. Всередині – Dockerfile, вихідний код,
конфігурація збірки та `docker-compose.yml` для локальної розробки.

Dockerfile є точкою входу для CI/CD – і ci-build.yml, і deploy.yml збирають
образ саме з нього.

## Docs

Проєктна документація. Не організаційна – організаційні стандарти живуть в
[.github](https://github.com/ChernihivPolytechnicNationalUniversity/.github)
репозиторії.

## Кореневі файли

- **README.md** – опис проєкту, tech stack, інструкції з запуску
- **.gitignore** – стандартний для стеку технологій
- **.env.example** – приклад змінних оточення для локальної розробки. `.env`
  файл не комітиться
