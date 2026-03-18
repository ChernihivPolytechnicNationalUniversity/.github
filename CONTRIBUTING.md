<div align="center">
  <img width="100%" src="https://capsule-render.vercel.app/api?type=waving&height=165&section=header&text=Contributing%20Guide&fontSize=48&fontColor=ffffff&animation=fadeIn&fontAlignY=50&theme=tokyonight" alt="Contributing Guide header" />
</div>
<img
  align="left"
  width="210"
  hspace="28"
  src="https://raw.githubusercontent.com/devicons/devicon/master/icons/git/git-original-wordmark.svg"
  alt="Git illustration"
/>

In this organization, contributors are expected to follow the **Conventional Commits**
specification when writing commit messages.

It defines a consistent commit format and supports structured metadata such as scopes,
breaking changes, and issue references. Some of these references can also integrate
directly with GitHub workflows, including issue linking and closing keywords.

<a href="https://www.conventionalcommits.org/en/v1.0.0/">
  <img src="https://img.shields.io/badge/Conventional%20Commits-Read%20the%20Specification-FE5196?style=for-the-badge&logo=git&logoColor=white" alt="Conventional Commits specification" />
</a>

<br clear="left" />


## Структура

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Type
- `feat`: Нова функціональність
- `fix`: Виправлення багу
- `docs`: Зміни тільки в документації
- `style`: Косметичні зміни, що не впливають на логіку
    - Пробіли, відступи, коми, форматування через Prettier/ESLint
- `refactor`: Зміна структури коду без зміни поведінки
- `test`: Додавання або виправлення тестів
- `chore`: catch-all для всього, що не підходить під інші типи
    - `.gitignore`, `.editorconfig`, конфіги лінтерів, ліцензійні хедери
- `perf`: Покращення продуктивності без зміни зовнішньої поведінки
- `ci`: Зміни в CI/CD конфігах
    - `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`
- `revert`: Відкат попереднього комміту
    - Стандартна практика: вказувати SHA комміту, що відкочується
- `build`: Зміни системи збірки або продакшн-залежностей
    - `package.json`, `webpack.config.js`, `Makefile`, `Dockerfile`, `gradle`

### Body
- Містить довільний текст
- Може складатися з будь-якої кількості параграфів з переносами рядків або просто аутлайн
- Потрібне для відповіді на питання: Чому?

### Footer
- Обов'язково складається з токена, після якого йде `: `, або `#`
    - `Token: значення`
    - `Token #значення`
- Токен також використовує `-`, але винятком є `BREAKING CHANGE`
- Може бути багаторядковим та з переносами рядків
    - Парсинг зупиняється, коли зустрічається новий токен
- Типи
    - `BREAKING CHANGE:` Зміна ламає зворотну сумісність
        - Будь-який тип комміту\*
            - ! Аналогом такого значення є `!`: `<type>[optional scope]!:`
                - На практиці використовуються разом
    - Всі вони закривають issue однаково (синоніми)
        - Підтримують вказування одразу декількох `Fixes #88, #91`
        - `Fixes #N`: *Цей комміт виправляє баг описаний в issue*
        - `Closes #N`: *Робота по цій задачі завершена*
            - Використовується для будь-чого, що не є багом
        - `Resolves #N`: *Цей комміт вирішує проблему або питання описане в issue*
    - `[Refs] #N`: Посилається на issue без закриття
        - *Цей комміт стосується тієї задачі, але не завершує її*
        - `Refs` чисто умовний текст
    - `Co-authored-by:` Позначає співавтора комміту
        - Формат обов'язковий: `Name <email@example.com>`
        - Якщо декілька, то з нового рядка вказувати знову з `Co-authored-by:`
    - `Acked-by:` *Я бачив ці зміни і підверджую їх*
        - Приклад: `Tech Lead <lead@example.com>`
