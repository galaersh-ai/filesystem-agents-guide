# Урок 3: Bash Tool

Создаём bash-инструмент — функцию `createBashTool` в `lib/tools.ts`, которая через Zod-схему описывает агенту, как выполнять команды в песочнице.

## Проблема

Вы строите файлового агента, а значит, агенту нужно объяснить, как использовать bash для навигации. LLM не «знает», какие команды ему доступны, — это нужно описать явно через инструмент. При этом инструмент должен сообщить модели, какие параметры он принимает (`command` и `args`), и безопасно выполнить их в песочнице.

## Решение

Определяем новый инструмент в `lib/tools.ts` — функцию, которая возвращает инструмент из AI SDK.

### Создание инструмента

Сначала определяем инструмент без выполнения в песочнице. Функция возвращает инструмент для выполнения bash-команд, но пока ничего не исполняет:

```typescript
// lib/tools.ts
import { tool } from 'ai';
import { z } from 'zod';

export function createBashTool() {
  return tool({
    description: `
      Execute bash commands to explore transcript and instruction files.
      Examples (not exhaustive): ls, cat, less, head, tail, grep
      `,
    inputSchema: z.object({
      command: z.string().describe('The bash command to execute'),
      args: z.array(z.string()).describe('Arguments to pass to the command')
    }),
    execute: async ({ command, args }) => {
      // code that executes when the tool is called
    }
  });
}
```

> **📝 Перевод промта:**
>
> *«Выполняй bash-команды для исследования файлов транскриптов и инструкций. Примеры (не исчерпывающие): ls, cat, less, head, tail, grep»*
>
> *«Команда bash, которую нужно выполнить» / «Аргументы, передаваемые команде»*

Пока колбэк `execute` пуст. Чтобы реально выполнять bash, который генерирует LLM, нужна безопасная среда выполнения — здесь на сцену выходит песочница.

### Принимаем песочницу как параметр

Чтобы передать экземпляр песочницы, обновляем объявление функции так, чтобы она ожидала песочницу в параметре:

```typescript
// lib/tools.ts
import type { Sandbox } from '@vercel/sandbox';

export function createBashTool(sandbox: Sandbox) {
  // ...
}
```

### Определяем, что делает инструмент

Передаём команды и аргументы в песочницу для выполнения:

```typescript
// lib/tools.ts
execute: async ({ command, args }) => {
  const result = await sandbox.runCommand(command, args);
  const textResults = await result.stdout();
  const stderr = await result.stderr();
  return {
    stdout: textResults,
    stderr: stderr,
    exitCode: result.exitCode,
  };
},
```

Через метод `runCommand` у `sandbox` вы выполняете сгенерированную команду и ожидаете стандартный вывод, вывод ошибок и код завершения процесса.

Теперь вы почти готовы отдать этот инструмент агенту. Но сначала нужно инициализировать песочницу в агенте, чтобы передать её в `createBashTool`.

## Ключевые выводы

- Через Zod-схему вы сообщаете агенту, что ему нужно сгенерировать `command` и `args` для передачи в инструмент.
- `.describe()` на каждом поле Zod даёт LLM контекст о том, что туда помещать.
- `sandbox.runCommand(command, args)` возвращает объект результата; `.stdout()` и `.stderr()` — асинхронные методы.
- `Sandbox` импортируется как type-only импорт, так как нужен только для сигнатуры функции.

## Попробуй сам

Инструмент пока не подключён к агенту — это будет в следующем уроке. Сейчас просто убедитесь, что файл компилируется:

1. Проверьте TypeScript-ошибки в редакторе. Если у `lib/tools.ts` нет красных подчёркиваний — типы корректны.
2. Проверьте экспорт. `createBashTool` должен быть именованным экспортом, который принимает `Sandbox` и возвращает инструмент.

!!! note "Пока без рантайм-теста"
    Вы не сможете протестировать инструмент в браузере, пока не подключите его к агенту в следующем уроке. Пока просто убедитесь, что он компилируется.
