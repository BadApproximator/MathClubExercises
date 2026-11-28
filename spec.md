# Техническая спецификация «Матклуба»

Документ фиксирует техническое устройство сайта: стек, архитектуру по Feature-Sliced Design (FSD), модель контента, правила импортов и соглашения по коду. Что делать и в каком порядке, описано в [docs/PLAN.md](docs/PLAN.md); как это устроено, описано здесь. Если план и спецификация расходятся, решение принимает владелец, после чего правится один из документов.

## 1. Стек и окружение

| Часть | Выбор | Примечание |
| --- | --- | --- |
| Генератор | Astro 7, режим `output: "static"` | Страницы без JS по умолчанию |
| Язык | TypeScript, `astro/tsconfigs/strict` | `typescript@^6`, не обновлять до 7, пока его не поддерживает `@astrojs/check` |
| Контент | MDX (`@astrojs/mdx`) + коллекции контента Astro | Схема полей на Zod (`astro/zod`) |
| Формулы | `remark-math` + `rehype-katex`, CSS `katex` | Рендер при сборке, `throwOnError: true` |
| Стили | Обычный CSS: переменные + `<style>` в компонентах Astro | Без препроцессоров, Tailwind и CSS-in-JS |
| JS в браузере | Обычный TS в `<script>` внутри `.astro` | UI-фреймворки (React, Preact и т. п.) не подключаются без отдельного решения |
| Шрифты | Literata, Inter, JetBrains Mono, KaTeX; раздаются со своего домена | Через `@fontsource*` или Fonts API Astro, выбирается на шаге 2.1 |
| Хостинг | Cloudflare Pages, `npm run build` → `dist/` | Превью-ссылка на каждый PR |
| Node | ≥ 22.23.4 (`.nvmrc` = 22) | |

Не используются: Prettier (решение владельца), база данных, серверный рендеринг, UI-фреймворки.

Команды:

```sh
npm run dev      # localhost:4321
npm run build    # astro check && astro build → dist/
npm run preview
npm run check    # astro check
npm run new      # (шаг 3.2) создать папку нового занятия из шаблона
```

Проверка любого изменения: `npm run build` без ошибок.

## 2. Архитектура: Feature-Sliced Design

### 2.1 Слои

Код в `src/` делится на слои FSD. Сверху вниз:

| Слой | Папка | Что лежит | Есть слайсы |
| --- | --- | --- | --- |
| app | `src/app/` | Базовый макет `<html>`, глобальные стили и токены, скрипт темы без мигания, подключение шрифтов и KaTeX | нет |
| pages | `src/views/` | Страница целиком: собирает виджеты под конкретный адрес | да |
| widgets | `src/widgets/` | Крупные самостоятельные блоки страницы: шапка, подвал, архив, пагинация занятий | да |
| features | `src/features/` | Действия пользователя, которые требуют JS: переключение темы, позже поиск, копирование LaTeX, выбор языка | да |
| entities | `src/entities/` | Сущности предметной области: занятие, тема. Типы, доступ к данным, отображение сущности | да |
| shared | `src/shared/` | Без предметной логики: UI-примитивы, i18n, конфиг сайта, утилиты | нет (только сегменты) |

Слой `processes` не используется (он устарел в FSD).

**Почему `views`, а не `pages`.** Папка `src/pages/` в Astro зарезервирована под маршрутизацию: любой `.astro`/`.ts` файл в ней становится адресом. Поэтому слой pages из FSD живёт в `src/views/`, а `src/pages/` служит только тонким слоем маршрутов (см. 2.4). По смыслу `views` — это ровно слой pages из FSD, все правила для него те же.

### 2.2 Правило импортов

1. Модуль импортирует только из слоёв **ниже** своего: `views → widgets → features → entities → shared`. `app` может импортировать всё.
2. Слайсы одного слоя **не импортируют друг друга** (виджет не тянет другой виджет, сущность не тянет другую сущность). Исключение для связи сущностей — кросс-импорт через `@x` (см. 2.5).
3. Снаружи слайса импортируется **только его публичный API** — `index.ts` в корне слайса. Глубокие импорты (`@entities/session/model/types`) запрещены.
4. Внутри слайса — относительные импорты (`./model/types`, `../lib/format`); алиасы слоёв внутри своего же слайса не используются.
5. `shared` и `app` не имеют слайсов, поэтому импорт из них идёт по сегменту: `@shared/i18n`, `@shared/ui/container`. У каждого такого модуля есть свой `index.ts`.

Алиасы в `tsconfig.json` (Vite подхватывает их автоматически):

```json
{
  "compilerOptions": {
    "paths": {
      "@app/*": ["./src/app/*"],
      "@views/*": ["./src/views/*"],
      "@widgets/*": ["./src/widgets/*"],
      "@features/*": ["./src/features/*"],
      "@entities/*": ["./src/entities/*"],
      "@shared/*": ["./src/shared/*"]
    }
  }
}
```

Проверка правил: пока вручную на ревью. Автоматическую проверку (линтер FSD `steiger` или ESLint-правила границ) можно добавить отдельным шагом по решению владельца; в его конфиге нужно исключить `src/pages/**` и указать, что слой pages — это `src/views`.

### 2.3 Сегменты внутри слайса

| Сегмент | Содержимое |
| --- | --- |
| `ui/` | Компоненты `.astro` и их стили |
| `model/` | Типы предметной области, чистые функции бизнес-логики (сортировка, соседние занятия, следующая тема) |
| `api/` | Доступ к данным: обращение к `astro:content`, превращение записей коллекций в модели |
| `lib/` | Вспомогательные функции, нужные только этому слайсу |
| `config/` | Константы слайса |

Пустые сегменты не создаются. Публичный API собирает нужное:

```ts
// src/entities/topic/index.ts
export type { Topic, TopicId } from "./model/types";
export { getTopics, getTopic } from "./api/get-topics";
export { getNextTopic } from "./model/cycle";
export { default as TopicBadge } from "./ui/TopicBadge.astro";
```

### 2.4 Маршруты `src/pages/` — тонкий адаптер

Файл маршрута делает только три вещи: описывает `getStaticPaths`, получает данные через API сущностей и рендерит страницу из `views`. Разметки, стилей и логики в нём нет.

```astro
---
// src/pages/sessions/[slug].astro
import { getSessions } from "@entities/session";
import { SessionPage } from "@views/session";

export async function getStaticPaths() {
  const sessions = await getSessions("ru");
  return sessions.map((session) => ({ params: { slug: session.slug }, props: { session } }));
}

const { session } = Astro.props;
---
<SessionPage session={session} />
```

Для другого языка появляется такой же тонкий файл `src/pages/en/sessions/[slug].astro` с `getSessions("en")`, а страница из `views` переиспользуется без изменений.

Служебные адреса (`rss.xml.ts`, картинки превью для соцсетей, `404.astro`) тоже лежат в `src/pages/` и тоже только вызывают код из нижних слоёв.

### 2.5 Связь сущностей: `@x`

Занятие ссылается на тему. Если модели занятия нужен тип или функция темы, тема открывает для этого отдельный узкий API:

```text
src/entities/topic/@x/session.ts   ← только то, что нужно сущности session
```

```ts
// в entities/session
import type { Topic } from "@entities/topic/@x/session";
```

Это единственный разрешённый импорт между слайсами одного слоя. Отображение «занятие вместе с меткой темы» собирается уровнем выше (в виджете), а не внутри сущности.

## 3. Структура проекта

```text
MathClubExercises/
  astro.config.mjs               ← интеграции, remark/rehype, i18n
  tsconfig.json                  ← strict + алиасы слоёв
  public/                        ← favicon и файлы как есть
  scripts/                       ← инструменты вне сайта: npm run new, сборка PDF (Pandoc)
  docs/PLAN.md
  spec.md
  src/
    content.config.ts            ← коллекции sessions и topics, схемы Zod
    content/                     ← данные, не код (вне слоёв FSD)
      topics.yaml
      sessions/
        012-wilson/
          index.mdx
          index.en.mdx           ← перевод, необязателен
          fig-1.svg
    pages/                       ← только маршруты Astro (2.4)
      index.astro
      about.astro
      404.astro
      sessions/index.astro
      sessions/[slug].astro
      topics/[topic].astro
      styleguide.astro           ← служебная, шаг 2.1
    app/
      layouts/BaseLayout.astro   ← <html>, <head>, мета, шрифты, KaTeX CSS, скрипт темы
      styles/
        tokens.css               ← цвета, шрифты, отступы, шкала заголовков
        themes.css               ← светлая и тёмная темы
        base.css                 ← сброс и типографика
    views/
      home/        ui/HomePage.astro
      session/     ui/SessionPage.astro
      archive/     ui/ArchivePage.astro
      topic/       ui/TopicPage.astro, model/collect-literature.ts
      about/       ui/AboutPage.astro
      not-found/   ui/NotFoundPage.astro
      styleguide/  ui/StyleguidePage.astro
    widgets/
      site-header/               ← логотип, меню, место под ThemeSwitch
      site-footer/
      session-hero/              ← шапка занятия: №, дата, тема, сложность, ссылки
      session-pager/             ← «← предыдущее / следующее →»
      related-sessions/          ← «ещё по теме»
      session-archive/           ← список по месяцам, lib/group-by-month.ts
      week-problem/              ← задача недели и домашняя на главной
      topic-tiles/               ← пять плиток тем
      topic-cycle/               ← полоса цикла: текущая и следующая тема
    features/
      theme-switch/              ← кнопка-островок, localStorage
      (позже) search/, copy-latex/, locale-switch/
    entities/
      session/
        model/types.ts           ← Session, Difficulty, LiteratureItem
        model/navigation.ts      ← соседние занятия, похожие по теме
        api/get-sessions.ts      ← getSessions, getSession, getLatestSession, renderSession
        ui/SessionCard.astro, DifficultyStars.astro, LiteratureList.astro
        ui/blocks/               ← MDX-блоки (раздел 5)
        index.ts
      topic/
        model/types.ts, model/cycle.ts
        api/get-topics.ts
        ui/TopicBadge.astro
        @x/session.ts
        index.ts
    shared/
      config/                    ← название сайта, базовый URL, список языков
      i18n/                      ← словари интерфейса, t(), локализация путей
      ui/                        ← Container, Prose, Icon, VisuallyHidden …
      lib/                       ← формат дат, склонения, общие утилиты
```

Структура показывает целевое состояние. Папки и слайсы создаются в том шаге плана, где они впервые нужны; заранее пустые папки не заводятся.

## 4. Модель контента

### 4.1 Коллекции

Коллекции описаны в `src/content.config.ts`, это единственный источник схем. Сущности не дублируют схему, а выводят типы из `CollectionEntry` и превращают записи в свои модели в сегменте `api`.

**topics** — загрузчик `file("src/content/topics.yaml")`, объект с ключом-идентификатором:

```yaml
number-theory:
  order: 1                       # место в цикле тем
  title: { ru: "Теория чисел" }  # en, sr добавляются позже
  description: { ru: "…" }
  color: { light: "#B5523B", dark: "#FF8B6B" }
```

Идентификаторы тем: `number-theory`, `linear-algebra`, `calculus`, `probability`, `geometry`.

**sessions** — загрузчик `glob({ base: "src/content/sessions", pattern: "*/index*.mdx" })` со своим `generateId`: идентификатор `<lang>/<папка>`, например `ru/012-wilson` для `index.mdx` и `en/012-wilson` для `index.en.mdx`.

| Поле | Тип | Правило |
| --- | --- | --- |
| `number` | целое > 0 | Совпадает с числом в имени папки, уникально |
| `title` | строка | Не пустая |
| `date` | дата | Обязательна, `z.coerce.date()` |
| `topic` | `reference("topics")` | Только существующая тема |
| `difficulty` | 1, 2 или 3 | |
| `links.video` | URL | Необязательно; кнопка показывается, только если заполнено |
| `links.post` | URL | Необязательно |
| `literature[]` | `{ author, title, note? }` | Необязательно |

Сообщения об ошибках схемы пишутся по-русски и называют поле и допустимые значения: «Поле topic: темы "algebra" нет в topics.yaml. Допустимо: number-theory, …». Проверки, которые Zod не умеет делать по всей коллекции (уникальность номера, совпадение номера с папкой), выполняет `entities/session/api` и при нарушении бросает ошибку с понятным текстом, которая останавливает сборку.

### 4.2 Модель занятия

```ts
// entities/session/model/types.ts
export type Difficulty = 1 | 2 | 3;

export interface Session {
  id: string;          // "ru/012-wilson"
  slug: string;        // "012-wilson" — часть адреса
  lang: Locale;
  number: number;
  title: string;
  date: Date;
  topicId: TopicId;
  difficulty: Difficulty;
  links: { video?: URL; post?: URL };
  literature: LiteratureItem[];
}
```

Слои выше не обращаются к `astro:content` напрямую: только через `getSessions`, `getSession`, `getLatestSession`, `renderSession` из `@entities/session` и `getTopics`, `getTopic` из `@entities/topic`. Если перевода занятия нет, `getSessions("en")` по умолчанию не включает его в список (поведение уточняется при включении второго языка).

## 5. Тело занятия: формулы и блоки

- Формулы: `$…$` и `$$…$$`. `rehype-katex` с `throwOnError: true` и `strict: "error"`: сломанная формула останавливает сборку. Выносные формулы обёрнуты в контейнер с `overflow-x: auto`.
- MDX-блоки лежат в `entities/session/ui/blocks/` и передаются в `<Content components={sessionBlocks} />`, поэтому в `.mdx` их не нужно импортировать:

| Блок | Разметка | Поведение |
| --- | --- | --- |
| `<Problem kind="main \| homework">` | `<section>` с полосой цвета темы | |
| `<Hint>` | `<details><summary>` | Свёрнут, раскрывается без JS и с клавиатуры |
| `<Solution>` | `<details><summary>` | То же; у домашней задачи добавляется через неделю |
| `<Remark>` | `<aside>` | Серый фон |
| `<Theorem kind="theorem \| definition \| lemma">` | `<div>` с курсивом | |
| `<Figure src caption>` | `<figure><figcaption>` | Номер «Рис. N» — CSS-счётчик в пределах статьи |

- Цвет темы попадает в блоки через CSS-переменную `--topic-color`, которую выставляет страница занятия. Сами блоки о темах ничего не знают.

## 6. Стили и темы

- Все цвета, шрифты, отступы и размеры задаются переменными в `app/styles/tokens.css`. В компонентах — только переменные, никаких «голых» HEX-значений.
- Значения палитры — из раздела «Типографика, цвета…» плана.
- Тема хранится в атрибуте `data-theme="light|dark"` на `<html>`. Без сохранённого выбора действует `prefers-color-scheme`.
- Без мигания: в `<head>` `BaseLayout` стоит маленький `<script is:inline>`, который до отрисовки читает `localStorage` и ставит `data-theme`. Сама кнопка — `features/theme-switch`.
- Стили компонента — в его `<style>` (scoped по умолчанию). Глобальные стили — только в `app/styles/`.
- Цвет темы занятия: из `topics.yaml` в разметку передаются обе пары (`--topic-light`, `--topic-dark`), а `themes.css` выбирает нужную в `--topic-color`.

## 7. JavaScript в браузере

- По умолчанию страница уходит без JS. Всё, что можно сделать на HTML и CSS (`<details>`, ссылки-фильтры), так и делается.
- Островок — это `.astro`-компонент в слое `features` с `<script>` на TypeScript. Astro сам собирает и подключает его только на страницах, где компонент используется.
- Островок работает поверх серверной разметки (прогрессивное улучшение): без JS страница остаётся читаемой.

## 8. Многоязычность

- `astro.config.mjs`: `i18n: { defaultLocale: "ru", locales: ["ru"], routing: { prefixDefaultLocale: false } }`. Новые языки (`en`, `sr`) дописываются в `locales` и в `shared/config`.
- Все надписи интерфейса — в `shared/i18n/dictionaries/ru.ts`. Остальные словари типизируются по русскому (`satisfies Dictionary`), чтобы пропущенный ключ ломал `astro check`.
- `t(locale)` возвращает функцию перевода; `localizePath(path, locale)` строит адрес с префиксом языка. Строки интерфейса прямо в разметке не пишутся.
- Перевод занятия — соседний файл `index.<lang>.mdx`, картинки общие.

## 9. Сборочные дополнения (этап 5)

| Задача | Где | Как |
| --- | --- | --- |
| PDF-листочек | `scripts/pdf/` + GitHub Actions | Pandoc с Lua-фильтром для MDX-блоков → Tectonic/XeLaTeX |
| Превью для соцсетей | `src/pages/og/[slug].png.ts` | Генерация картинки при сборке; общий рендер в `shared/lib/og-image`, данные из `@entities/session` |
| RSS | `src/pages/rss.xml.ts` | `@astrojs/rss` |
| Карта сайта | `astro.config.mjs` | `@astrojs/sitemap`, нужен `site` |
| `npm run new` | `scripts/new-session.ts` | Следующий номер, папка `NNN-slug`, шаблон `index.mdx` |

Скрипты в `scripts/` не входят в слои FSD и не импортируются из `src/`.

## 10. Соглашения по коду

- Имена: слайсы и сегменты — `kebab-case`; компоненты — `PascalCase.astro`; прочие файлы — `kebab-case.ts`; типы — `PascalCase`; функции — `camelCase` с глаголом (`getSessions`, `groupByMonth`).
- Пропсы компонента описываются `interface Props` в его frontmatter.
- `any` запрещён; `as` — только с комментарием, почему иначе нельзя.
- Даты форматируются только через `shared/lib` с учётом языка.
- Отступы, переводы строк и кодировка — по `.editorconfig` (2 пробела, LF, UTF-8).
- Комментарии в коде — по-русски и только там, где неочевидно «почему».
- Тестов пока нет. Когда появятся, тестируются чистые функции из `model/` и `lib/` (Vitest), файлы тестов лежат рядом: `navigation.test.ts`.

## 11. Открытые технические вопросы

1. **Задача недели на главной (шаг 4.4).** Условия задач находятся в теле MDX, а главной нужен их готовый HTML. Предложение: remark-плагин `entities/session/lib/remark-extract-problems.ts` достаёт содержимое `<Problem>` и кладёт его в `remarkPluginFrontmatter`. Запасной вариант — отдельное поле во frontmatter, но тогда текст дублируется.
2. **Шрифты (шаг 2.1).** `@fontsource` или встроенный Fonts API Astro: решается по тому, что стабильно в установленной версии.
3. **Автопроверка границ FSD.** Подключать ли `steiger` и когда.
4. **Занятие без перевода на втором языке.** Скрывать или показывать русскую версию с пометкой.
