---
title: "مرتب سازی خودکار کدها در زمان Commit"
categories:
  - Net
tags:
  - net
  - git
  - format
---

با استفاده از کتابخانه Husky.Net می‌توانید تنظیماتی انجام دهید که در زمان کار با Git یک سری کارها بصورت خودکار انجام شود.  
بطور مثال در این مطلب ما در زمان هر Commit دستور dotnet format اجرا شود تا کدهای تغییر کرده بصورت خودکار مرتب شوید.  
برای این کار ابتدا دستور زیر را در بیس پروژه جایی که .git وجود دارد در CMD بزنید:  

 > `dotnet new tool-manifest`

سپس در همان محل دستور زیر را وارد نمایید:  

 > `dotnet tool install Husky`

با این کار فایل زیر در پوشه `.config` اجرا می‌شود.  

```
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "husky": {
      "version": "0.7.2",
      "commands": [
        "husky"
      ],
      "rollForward": false
    }
  }
}
```

همچنین با زدن دستور دوم یک پوشه `.Husky` نیز درست می‌شود. پیشنهاد می‌شود فایل `task-runner.json` را بصورت زیر تغییر دهید و همچنین فایل `schema.json` را نیز دانلود و در همان محل قرار دهید.  

```
{
  "$schema": "schema.json",
  "tasks": []
}
```

فایل schema.json
```
{
   "$schema": "http://json-schema.org/draft-07/schema#",
   "title": "TaskRunner",
   "type": "object",
   "properties": {
     "tasks": {
       "type": "array",
       "items": {
         "$ref": "#/definitions/huskyTask"
       },
       "description": "A list of tasks that the runner will execute. Each task is defined with specific commands and configurations."
     },
     "variables": {
       "type": "array",
       "items": {
         "$ref": "#/definitions/variableTask"
       },
       "description": "A list of variable tasks that can override default settings or provide new ones."
     }
   },
   "required": ["tasks"],
   "definitions": {
     "huskyTask": {
       "type": "object",
       "additionalProperties": false,
       "properties": {
         "name": {
           "type": "string",
           "minLength": 1,
           "description": "The name of the task, recommended for identification."
         },
         "command": {
           "type": "string",
           "minLength": 1,
           "description": "Path to the executable file, script, or executable name to run."
         },
         "args": {
           "type": "array",
           "items": {
             "type": "string"
           },
           "description": "Array of command arguments. Built-in variables can be used, such as ${staged}, ${git-files}, ${last-commit}",
           "examples": ["${staged}", "${git-files}", "${last-commit}", "${args}", "${all-files}"]
         },
         "output": {
           "$ref": "#/definitions/outputTypes",
           "description": "Specifies the output log level. Can be 'always', 'verbose', or 'never'.",
           "default": "always"
         },
         "pathMode": {
           "$ref": "#/definitions/pathModes",
           "description": "Defines the file path style. Can be 'relative' or 'absolute'.",
           "default": "relative"
         },
         "cwd": {
           "type": "string",
           "description": "Current working directory for the command.",
           "default": "."
         },
         "group": {
           "type": "string",
           "description": "Group of the task, usually the hook name."
         },
         "branch": {
           "type": "string",
           "description": "Regex to run the task on specific branches only."
         },
         "include": {
           "type": "array",
           "items": {
             "type": "string"
           },
           "description": "Glob pattern to select files."
         },
         "exclude": {
           "type": "array",
           "items": {
             "type": "string"
           },
           "description": "Glob pattern to exclude files."
         },
          "filteringRule": {
             "$ref": "#/definitions/filteringRules",
             "description": "The filtering rule for this task. Can be 'variable' or 'staged'.",
             "default": "variable"
          },
         "windows": {
           "$ref": "#/definitions/windowsOverrides",
           "description": "Overrides all settings for Windows."
         }
       },
       "required": ["name", "command"]
     },
     "windowsOverrides": {
       "type": "object",
       "additionalProperties": false,
       "properties": {
         "name": {
           "type": "string",
           "minLength": 1,
           "description": "Override task name for Windows."
         },
         "command": {
           "type": "string",
           "minLength": 1,
           "description": "Override command for Windows."
         },
         "args": {
           "type": "array",
           "items": {
             "type": "string"
           },
           "description": "Override arguments for Windows. Built-in variables can be used, such as ${staged}, ${git-files}, ${last-commit}",
           "examples": ["${staged}", "${git-files}", "${last-commit}"]
         },
         "output": {
           "$ref": "#/definitions/outputTypes",
           "description": "Override output log level for Windows.",
           "default": "always"
         },
         "pathMode": {
           "$ref": "#/definitions/pathModes",
           "description": "Override path mode for Windows.",
           "default": "relative"
         },
         "cwd": {
           "type": "string",
           "description": "Override working directory for Windows.",
           "default": "."
         },
         "group": {
           "type": "string",
           "description": "Override group for Windows."
         },
         "branch": {
           "type": "string",
           "description": "Override branch for Windows."
         },
         "include": {
           "type": "array",
           "items": {
             "type": "string"
           },
           "description": "Override include pattern for Windows."
         },
         "exclude": {
           "type": "array",
           "items": {
             "type": "string"
           },
           "description": "Override exclude pattern for Windows."
         }
       }
     },
     "variableTask": {
       "type": "object",
       "additionalProperties": false,
       "properties": {
         "name": {
           "type": "string",
           "minLength": 1,
           "description": "The name of the variable task."
         },
         "command": {
           "type": "string",
           "minLength": 1,
           "description": "Command for the variable task."
         },
         "args": {
           "type": "array",
           "items": {
             "type": "string"
           },
           "description": "Arguments for the variable task command. Built-in variables can be used, such as ${staged}, ${git-files}, ${last-commit}",
           "examples": ["${staged}", "${git-files}", "${last-commit}"]
         },
         "windows": {
           "$ref": "#/definitions/variableTaskOverrides",
           "description": "Overrides for the variable task on Windows."
         }
       },
       "required": ["name", "command"]
     },
     "variableTaskOverrides": {
       "type": "object",
       "additionalProperties": false,
       "properties": {
         "name": {
           "type": "string",
           "minLength": 1,
           "description": "Override task name for Windows."
         },
         "command": {
           "type": "string",
           "minLength": 1,
           "description": "Override command for Windows."
         },
         "args": {
           "type": "array",
           "items": {
             "type": "string"
           },
           "description": "Override arguments for Windows. Built-in variables can be used, such as ${staged}, ${git-files}, ${last-commit}",
           "examples": ["${staged}", "${git-files}", "${last-commit}"]
         }
       }
     },
     "outputTypes": {
       "type": "string",
       "enum": ["always", "verbose", "never"],
       "description": "Specifies the output log level.",
       "default": "always"
     },
     "pathModes": {
       "type": "string",
       "enum": ["relative", "absolute"],
       "description": "Specifies the file path style.",
       "default": "relative"
     },
     "filteringRules": {
         "type": "string",
         "enum": ["variable", "staged"],
         "default": "variable",
         "description": "The filtering rule for this task. Can be 'variable' or 'staged'."
      }
   }
 }
```

اکنون با دستور زیر کافی است یک فایل برای دستورات خود اجرا کنید:  

 > `dotnet husky add pre-commit -c "dotnet format"`

سپس کافی است محتویات آن را با دستوران زیر تغییر بدهید:  

```
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# Exit early on CI/CD
if [ "$CI" = "true" ] || [ "$TF_BUILD" = "true" ]; then
  exit 0
fi

# Get staged C# files (paths relative to repo root)
staged_files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.cs$' || true)

# Exit if no C# files staged
[ -z "$staged_files" ] && exit 0

# Run dotnet format on solution with staged files
dotnet format src/prj/My.sln --no-restore --include $staged_files

# Re-add formatted files to commit
# git add $staged_files
```

در دستور بالا ابتدا چک می‌شود تا دستور فوق در CI/CD اجرا نشود.  
سپس لیست فایل‌هایی که تغییر کرده دریافت می‌شود تا نیاز نباشد بر روی تمام فایل‌ها دستور مرتب سازی را اعمال کنیم.  
سپس با دستور `dotnet format` عملیات مرتب‌سازی انجام می‌شود.  
در این دستور با `--no-restore` جلوی بیلد و نوگت ریستور اضافه گرفته شده است.  
همچنین نیاز است آدرس `sln` نیز در آن مشخص شود.  

در صورتی که دستور آخر را از کامنت خارج کنید مرتب‌سازی اعمال شده در همان کامیت زده شده و می‌رود اما اگر کامیت باشد تغییرات انجام شده را می‌توانید مشاهده کنید و سپس بصورت جدا کامیت کنید.  