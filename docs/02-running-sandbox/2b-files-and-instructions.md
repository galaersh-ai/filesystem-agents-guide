# Урок 5: Files and Instructions

Загружаем транскрипты звонков в песочницу и добавляем агенту инструкции, чтобы он использовал bash-инструмент для исследования файлов и ответов на вопросы.

## Проблема

У вас есть 3 демо-транскрипта звонков в директории `lib/calls`, которые нужно загрузить как контекст в песочницу перед тем, как задать агенту вопрос. Агенту также нужны чёткие инструкции, чтобы он знал: использовать bash-инструмент для исследования файлов и поиска ответов.

## Решение

Дополняем `lib/agent.ts` двумя вещами: функцией загрузки файлов и строкой инструкций.

### Запись файлов в песочницу

Создаём функцию `loadSandboxFiles`, которая загружает файлы звонков в песочницу:

```typescript
// lib/agent.ts
import path from 'path';
import fs from 'fs/promises';

async function loadSandboxFiles(sandbox: Sandbox) {
  const callsDir = path.join(process.cwd(), 'lib', 'calls');
  const callFiles = await fs.readdir(callsDir);

  for (const file of callFiles) {
    const filePath = path.join(callsDir, file);
    const buffer = await fs.readFile(filePath);
    await sandbox.writeFiles([{ path: `calls/${file}`, content: buffer }]);
  }
}
```

Функция строит полный путь к папке `calls` (в той же директории `lib`, что и `agent.ts`), читает все имена файлов в ней, загружает каждый файл как буфер и записывает их в файловую систему песочницы.

Обязательно вызываем функцию до создания агента:

```typescript
// lib/agent.ts
await loadSandboxFiles(sandbox);
```

### Инструкции агенту

Теперь, когда всё подключено, пишем чёткие инструкции. Мы хотим, чтобы агент использовал bash-инструмент для генерации и выполнения команд, исследуя все транскрипты звонков и отвечая на вопросы пользователя:

```typescript
// lib/agent.ts
const INSTRUCTIONS = `
You are a helpful assistant that answers questions about customer calls. Use bashTool to explore the files and find relevant information pertaining to the user's query. Using the information you find, craft a response for the user and output it as text.
`;
```

> **📝 Перевод промта:**
>
> *«Ты — полезный ассистент, который отвечает на вопросы о звонках клиентов. Используй bashTool, чтобы исследовать файлы и найти информацию, относящуюся к запросу пользователя. На основе найденной информации составь ответ для пользователя и выведи его в виде текста.»*

Добавляем `INSTRUCTIONS` агенту:

```typescript
// lib/agent.ts
export const agent = new ToolLoopAgent({
  instructions: INSTRUCTIONS,
  // ...
});
```

### Полное решение

Вот полный файл `lib/agent.ts` со всеми частями вместе:

```typescript
// lib/agent.ts
import { ToolLoopAgent } from 'ai';
import { createBashTool } from './tools';
import { Sandbox } from '@vercel/sandbox';
import path from 'path';
import fs from 'fs/promises';

const INSTRUCTIONS = `
You are a helpful assistant that answers questions about customer calls. Use bashTool to explore the files and find relevant information pertaining to the user's query. Using the information you find, craft a response for the user and output it as text.
`;

const sandbox = await Sandbox.create();

const MODEL = 'anthropic/claude-opus-4.6';

await loadSandboxFiles(sandbox);

export const agent = new ToolLoopAgent({
  model: MODEL,
  instructions: INSTRUCTIONS,
  tools: {
    bashTool: createBashTool(sandbox)
  }
});

async function loadSandboxFiles(sandbox: Sandbox) {
  const callsDir = path.join(process.cwd(), 'lib', 'calls');
  const callFiles = await fs.readdir(callsDir);

  for (const file of callFiles) {
    const filePath = path.join(callsDir, file);
    const buffer = await fs.readFile(filePath);
    await sandbox.writeFiles([{ path: `calls/${file}`, content: buffer }]);
  }
}
```

Агент готов! Чат-интерфейс и API-маршрут для запуска агента и стриминга результатов уже созданы.

## Ключевые выводы

- Используйте `process.cwd()` для построения пути к `lib/calls/`.
- `fs.readdir` возвращает имена файлов, `fs.readFile` возвращает буфер.
- `sandbox.writeFiles` принимает массив объектов `{ path, content }`.
- Функцию `loadSandboxFiles` можно объявить после экспорта агента (hoisting), но вызывать её нужно **до** экспорта.
- Инструкции должны явно говорить агенту использовать `bashTool` — иначе он не догадается исследовать файлы.

## Попробуй сам

1. Перезапустите dev-сервер и откройте `http://localhost:3000`.
2. Спросите: **«what files are available?»** — агент должен вызвать `bashTool` с `ls calls/` и сообщить о трёх файлах:

```
[bashTool] $ ls calls/
1.md  2.md  3.md
```

3. Спросите: **«summarize the first call»** — агент должен выполнить `cat calls/1.md` (или похожее) и выдать сводку: участники, обсуждаемые темы, итоги.
4. Спросите: **«did anyone mention pricing?»** — агент должен выполнить `grep` по файлам на предмет упоминания цен и синтезировать ответ.

!!! success "Агент работает!"
    Если агент перечисляет файлы, читает транскрипты и отвечает на вопросы — вы построили работающего файлового агента. Следующий урок — о стратегиях тестирования и идеях расширения.
