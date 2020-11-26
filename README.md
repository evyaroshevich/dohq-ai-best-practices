# DevSecOps: внедрение в продуктовый конвейер и эксплуатация PT Application Inspector 
Узнать подробнее о внедрении PT Application Inspector можно из вебинара: [https://www.ptsecurity.com/ru-ru/research/webinar/devsecops-vnedrenie-v-produktovyj-konvejer-i-ehkspluataciya-pt-application-inspector/](https://www.ptsecurity.com/ru-ru/research/webinar/devsecops-vnedrenie-v-produktovyj-konvejer-i-ehkspluataciya-pt-application-inspector/)

_Инструкция ниже, методика и рекомендации актуальны для AIE сервера v.3.6.1_

# Содержание

1. [Для чего нужен PT Application Inspector](#для-чего-нужен-pt-application-inspector)
2. [Методика внедрения PT AI в CI](#методика-внедрения-pt-ai-в-ci)
3. [Описание сборочной CI-инфраструктуры](#описание-сборочной-ci-инфраструктуры)
4. [Автоматизация установки AIE сервера через PowerShell](#автоматизация-установки-aie-сервера-через-powershell)
5. [Цикл релизной сборки Application Inspector Shell Agent в Docker](#цикл-релизной-сборки-application-inspector-shell-agent-в-docker)
6. [Алгоритм работы методики с использованием AISA](#алгоритм-работы-методики-с-использованием-aisa)
7. [Типовые шаблоны для отправки проектов на сканирование в AIE](#типовые-шаблоны-для-отправки-проектов-на-сканирование-в-aie)
8. [Утилиты и переменные окружения](#утилиты-и-переменные-окружения)
9. [Инструкция по использованию метараннеров в TeamCity](#инструкция-по-использованию-метараннеров-в-teamcity)
10. [Содержимое репозитория](#содержимое-репозитория)
                                                                                                
## Для чего нужен PT Application Inspector

PT Application Inspector — удобный инструмент для выявления уязвимостей и ошибок в приложениях, поддерживающий процесс безопасной разработки.

PT Application Inspector выделяется среди конкурентов исключительной точностью результатов благодаря сочетанию ключевых методов анализа с уникальной технологией абстрактной интерпретации. PT AI позволяет специалистам по ИБ выявлять и подтверждать уязвимости и признаки НДВ, например закладок, оставленных в исходном коде разработчиками или хакерами, а разработчикам — ускорить исправление кода на ранних стадиях разработки.

**Список поддерживаемых языков:** `java`, `php`, `c#`, `vb`, `objective-c`, `c++`, `sql`, `swift`, `python`, `javascript`, `go`

Подробнее о продукте: [https://www.ptsecurity.com/ru-ru/products/ai/](https://www.ptsecurity.com/ru-ru/products/ai/)

## Методика внедрения PT AI в CI

**Основные этапы методики внедрения PT AI в CI:**

1. Подготовка серверной части
   - Установка и настройка сервера
   - Установка и настройка агентов сканирования
2. Подготовка клиентской части
   - Организация релизного CI-цикла для клиента AISA (поставка docker-образом)
3. Подготовка проекта сканирования
   - На стороне сервера
   - Через клиент AISA
4. Работа с CI-системами
   - Шаблоны сканирования в GitLab CI
   - Метараннеры TeamCity
   - Прочие средства (работа с CLI AISA)

![](/.media/01_AI-Arch.png)

**Легенда**

- DevOps.BuildAgent - сборочный агент (Linux или Windows)
- BuildAgent.Console - системная консоль сборочного агента
- Server.AIE.Agent - агент сканирования (PT Application Inspector)
- DevOps.GitLab - корпоративное хранилище кода
- DevOps.GitLab-CI - корпоративная CI-система
- DevOps.Artifactory - корпоративное хранилище артефактов (Артифакторий)
- Docker.Registry - хранилище докер-образов в Артифактории
- DevOps.Artifactory.Repo - репозиторий в Артифактории для хранения бинарных файлов, выгружаемых туда после сборки
- Docker.Build.AISA - артефакт сборки клиента AISA (докер-образ)
- Docker.Windows/Linux.AISA-client.Latest/TAG - все докер-образы клиента AISA (Linux и Windows)
- Docker.Windows/Linux.AISA-client - выбранный докер-образ, "обёртка" над клиентом AISA
- AIE.LightweightClient - легковесный клиент AISA внутри докер-контейнера для работы с API сервера PT Application Inspector

Мы рекомендуем развернуть серверную часть AI (на схеме обозначено как Server.AIE.Agent) и сканирующие агенты на отдельных ВМ.

Исходный код (source code) из проекта на GitLab (DevOps.GitLab) выгружается на сборочный агент (DevOps.BuildAgent) в рабочую директорию сборки, а затем передаётся для анализа на сервер AI через консольный клиент AISA (AIE.LightweightClient). Клиент умеет работать с API сервера AI. AISA — это аббревиатура от Application Inspector Shell Agent.

Возможности передать код на AI-сервер напрямую из GitLab-проекта, без физического копирования со сборочного агента, сейчас нет, но в будущих версиях AISA такая возможность появится.

Клиент AISA запускается в отдельном докер-контейнере (Docker.Windows/Linux.AISA-client), который является своеобразной "обёрткой" над ним. Образы собираются служебными сборочными конфигурациями (DevOps.GitLab-CI), которые поддерживают CI-инженеры нашего DevOps-отдела. После сборки образы выкладываются в docker registry на Артифактории (DevOps.Artifactory). Рекомендуется настроить релизный цикл и приёмочное тестирование для подготовки таких образов, как показано далее. Благодаря поставке докер-образами сборочная инфраструктура не будет зависеть от разработки клиента AISA.

**Плюсы архитектуры:**

- Минимальные трудозатраты инженеров группы инфраструктуры на развёртывание, эксплуатацию и обновление сервера AI, в отличие от двух других вариантов архитектур.
- Требуется настройка мониторинга только сервера AI и сканирующих агентов, то есть небольшого количества ВМ.
- Наличие API: команды для передачи кода на сканирование унифицированы через легковесный клиент AISA и не придётся перенастраивать сотни сборочных конфигураций в случае изменения контрактов в новых версиях AI-сервера.
- По аналогичному принципу (передача кода на анализ во внешние сервисы) работают все облачные сканеры кода, например, Codacy и SonarQube. Хотя GitLab использует интегрированное серверное решение для своего Code Quality сервиса.
- В предлагаемой архитектуре можно отделить анализ кодовой базы от сборочного процесса.

**Минусы архитектуры:**

- Более сложное внедрение анализатора кода в сборочный процесс, за счёт дополнительных шагов и настроек сборки. Требуется разработка скриптов интеграции с CI-системами и шаблонов для запуска сканирования. Это сложнее, чем в случае, когда AI-сервер и агент сканирования расположены на билд-агенте.
- Потенциально может возникнуть больше проблем со сборками, за счёт добавления нового внешнего сервиса в сборочный конвейер.
- Возможно значительное ожидание сборок в очереди, пока сканирование идёт на одном AI-сервере.
- Пока неизвестны требования по железу для масштабирования и обеспечения сканирований для десятков и сотен тысяч сборок в день.

Как мы решали некоторые проблемы с выбранной архитектурой, смотрите [демо на вебинаре](https://www.ptsecurity.com/ru-ru/research/webinar/devsecops-vnedrenie-v-produktovyj-konvejer-i-ehkspluataciya-pt-application-inspector/).

## Описание сборочной CI-инфраструктуры

![](/.media/03_idef0-devops-process-habr.png)

Особое внимание мы уделяем разработке типовых проектов для систем непрерывной интеграции. Мы выделяем так называемую релизную схему сборок с продвижениями, основную часть которой вы видите на схеме. Если очень упростить и обобщить эту релизную схему, то она включает в себя следующие этапы:

- кроссплатформенная сборка продукта,
- деплой на тестовые стенды,
- выполнение функциональных и иных тестов,
- продвижение протестированных сборок в релизные репозитории на Artifactory,
- публикация релизных сборок на серверах обновлений,
- доставка сборок и обновлений на инфраструктуру заказчиков,
- запуск инсталляции или обновления продукта.

Скорее всего, в вашей компании присутствуют примерно такие же этапы разработки. Встаёт задача: **как внедрить в уже имеющиеся и сложившиеся сборочные процессы новый этап сканирования кода при помощи Application Inspector**? Мы рассматривали несколько вариантов:

- (перед Promoting) выполнять сканирование кода только протестированных компонент перед их продвижением в релизный репозиторий,
- (перед Publishing) выполнять сканирование кода только релизных инсталляторов и их компонент перед их публикацией на серверах обновлений,
- (внутри Testing) сделать сканирование кода как один из видов приёмочных тестов,
- (перед Building) запускать сканирование кода до выполнения шага компиляции компоненты на этапе сборки,
- (после Building) запускать сканирование кода после компиляции компоненты, перед выкладкой артефакта в хранилище.

Каждый из этих вариантов имеет плюсы и минусы: например, сканируя код только перед продвижением в релиз можно минимизировать нагрузку на сканирующий сервер AI. Однако мы рекомендуем вам вариант: **"запускать сканирование кода до выполнения шага компиляции компоненты на этапе сборки"**.

Этот вариант позволяет достаточно легко внедрить шаг сканирования прямо в шаблоны сборочных конфигураций и тиражировать это решение по множеству компонент и инсталляторов продуктов. Для этого мы написали несколько скриптов для запуска сканирования, типовые задачи (job) для GitLab CI, с примерами параметризации и запуска этих скриптов, и сохранили их в этом репозитории.

Таким образом, вы можете избавить разработку от необходимости обращаться к инженерам DevOps: при необходимости они сами могут подключить шаги сканирования в сборках новых компонент.

![](/.media/02_AI-in-build-steps.png)

## Автоматизация установки AIE сервера через PowerShell

Для облегчения установки AIE сервера можно использовать автоматический установщик, исходный код которого доступен в директории [AIE_Server_installation](/AIE_Server_installation)

## Цикл релизной сборки Application Inspector Shell Agent в Docker

![](/.media/04_PlantUML_Docker_Build.png)
![](/.media/06_TeamCity_CI_Process.png)

### Инструкция по cборке Docker контейнеров

- [ ] Добавить комментарии, что как временное решение используются вспомогательные утилиты (скрипты на Python)
- [ ] Описать, откуда можно взять установчные бинари для клиента
    
## Алгоритм работы методики с использованием AISA

![](/.media/05_PlantUML_Ci_Process.png)
- [ ] Пошагово расписать как выполняется job'a на примере схемы
- [ ] Параметризацию проекта
- [ ] Добавить схему задачи на сканирование
- [ ] Описать каким образов сделать параметризацию проекта (временное решение)
    
## Типовые шаблоны для отправки проектов на сканирование в AIE
### Как подключить AIE скан в сборочном проекте GitLab CI
- Выберите шаблон в директории [GitLab_templates](/GitLab_templates).
- Откройте `.gitlab-ci.yml` в корне вашего сборочного проекта.
- Добавьте `include` выбранного вами шаблона.
- При необходимости добавьте необходимые `Variables` в `.gitlab-ci.yml` вашего сборочного проекта.
- В случае, если ваш проект сканируется слишком продолжительное время, вы можете сканировать его в режиме без ожидания с помощью аргумента `--no-wait`

Пример добавления шаблона вы можете посмотреть тут: [example.yml_1](/GitLab_templates/example_1.yml)и [example_2.yml](/GitLab_templates/example_2.yml)

### Какие типовые шаблоны существуют

###### Шаблон для **существующего** проекта на AIE сервере
- Имя проекта задается автоматически как `$CI_PROJECT_NAME` (имя проекта в GitLab).
- Отчет создается автоматически в `HTML` и `JSON` форматах и складывается в дирректорию `.report`. Доступен на протяжении трех дней.

###### Шаблон для **не существующего** проекта на AIE сервере
- Имя проекта задается вручную (один раз на шаблон).
- Язык программирования задается дополнительной переменной.
- Путь к `sln` файлу (для `VB` и `csharp` проектов) задается дополнительной переменной.
- Отчет получается в виде артефакта в `HTML` и `JSON` форматах и складывается в дирректорию `.report`. Доступен на протяжении трех дней.

###### Продуктовые шаблоны
- Создаются под нужды конкретной команды (могут быть типовыми для большого количества проектов).
- Широкая возможность кастомизации.
- Команда может вносить правки самостоятельно и создавать свои шаблоны (через **MR**).

### Как получить отчет
Существует несколько вариантов получения отчета:
1. Обратившись к AIE серверу напрямую.
2. Получить в виде артефакта после завершения сканирования.

Отчет сохраняется на GitLab раннере в виде артефакта и может одновременно быть в нескольких форматах: `HTML`, `PDF`, `JSON` и `WAF`. Отчет может быть интегрирован с различными системами и сервисами. Например, он может быть:
- сохранен на Artifactory,
- интегрирован с GitLab Code Quality в MR,
- опубликован в логах сборки GitLab,
- опубликован на GitLab Pages,
- отправлен на e-mail,
- etc.

## Утилиты и переменные окружения

Docker образ с aisa поставляется совместно со следующими утилитами:
- `aisa`— утилита для отправки исходного кода на AIE сервер и получения отчета,
- `aisa-set-settings` — утилита позволяет параметризовать настройки проекта перед отправкой на AIE сервер,
- `aisa-set-policy` — утилита позволяет параметризовать настройки политики сканирования перед отправкой на AIE сервер,
- `aisa-codequality` — утилита позволяет обработать полученный в ходе отчета `JSON` для его конвертации в читаемый для GitLab'a вид

### aisa
| Параметр запуска          | Пример значения | Обязательный параметр?                                  | Описание                                                                                                                                                                           |
|---------------------------|-----------------|---------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `--project-name`          | `Test`          | Да, при отсутствии ключа `--project-settings-file`      | Название проекта, который надо просканировать (регистронезависимо). Проект должен быть создан на момент запуска.                                                                   |
| `--project-settings-file` | `Test.aiproj`   | Да, при отсутствии ключа `--project-name`               | Путь к файлу с настройками проекта.                                                                                                                                                |
| `--scan-target`           | `./`            | Да                                                      | Путь к папке или файлу приложения для сканирования.                                                                                                                                |
| `--reports-folder`        | `reports`       | Нет                                                     | Путь к папке, куда будут сохранены файлы отчетов.                                                                                                                                  |
| `--reports`               | `"HTML,JSON"`   | Нет                                                     | Типы отчетов, создаваемых по окончании сканирования. Могут принимать одно или несколько из следующих значений: HTML, PDF, JSON, WAF. Для нескольких значений используется запятая  |
| `--policies-path`         | `policy.json`   | Нет                                                     | Путь к файлу с описанием политик.                                                                                                                                                  |
| `--no-wait`               |                 | Нет                                                     | Позволяет запустить сканирование в режиме без ожидания получения результатов                                                                                                       |
| `--scan-off`              |                 | Нет, работает только с ключем `--project-settings-file` | Позволяет не запускать сканирование в случае, если мы создаем проект через файл с настройками проекта.                                                                             |

### aisa-set-settings
| Параметр запуска | Пример значения  | Обязательный параметр? | Описание                                                         |
|------------------|------------------|------------------------|------------------------------------------------------------------|
| `--projectname`  | `DevOps_Sandbox` | Да                     | Название проекта (регистронезависимо)                            |
| `--language`     | `Python`         | Да                     | Язык программирования                                            |
| `--path`         | `./`             | Нет                    | Путь к дирректории с исходным кодом (по умолчанию "./")          |
| `--incl`         | `True`           | Нет                    | Оставить исключения включенными (True/False) по умолчанию `True` |

Список форматов, попадающих под исключение сканирования: `*.7z`, `*.bmp`, `*.dib`, `*.dll`, `*.doc`, `*.docx`, `*.exe`, 
`*.gif`, `*.ico`, `*.jfif`, `*.jpe`, `*.jpe6`, `*.jpeg`, `*.jpg`, `*.odt`, `*.pdb`, `*.pdf`, `*.png`, `*.rar`, `*.swf`, 
`*.tif`, `*.tiff`, `*.zip`

### aisa-set-policy
| Параметр запуска       | Пример значения | Обязательный параметр? |
|------------------------|-----------------|------------------------|
| `--count_to_actualize` | `1`             | Нет                    |
| `--level`              | `"Medium"`      | Нет                    |
| `--exploit`            | `'"."'`         | Нет                    |
| `--is_suspected`       | `"false"`       | Нет                    |
| `--approval_state`     | `"[^2]"`        | Нет                    |

### aisa-codequality
| Параметр запуска        | Пример значения | Обязательный параметр? |
|-------------------------|-----------------|------------------------|
| `-i` / `--input_folder` | `.report`       | Да                     |
| `-o` / `--output_file`  | `.report.json`  | Да                     |     


## Инструкция по использованию метараннеров в TeamCity

Установите метараннеры из каталога [TeamCity_meta-runners](/TeamCity_meta-runners) стандартным образом по инструкции [https://www.jetbrains.com/help/teamcity/working-with-meta-runner.html#Installing+Meta-Runner](https://www.jetbrains.com/help/teamcity/working-with-meta-runner.html#Installing+Meta-Runner)

## Содержимое репозитория

```
├── AIE_Server_installation                      # Директория с инструкциями по установки серверной части AIE
│   ├── AI-one-click-install.ps1                 # PowerShell скрипт для авто-установки серверной части AIE
│   └── README.md                                # Инструкция по автоматической установке серверной части AIE
├── AISA_Docker                                  # Директория с примерами сборки docker образов под Windows и Linux 
│   ├── aisa-linux                               # Директория с примерами сборки docker образа под Linux 
│   │   ├── docker_build_data                    # Директория со скриптами для сборки Docker
│   │   │   ├── applications                     # Директория со скриптами, импортируемыми в виде приложений 
│   │   │   │   ├── aisa-codequality.py          # Python скрипт для настройки Code Quality в GitLab-CI
│   │   │   │   ├── aisa-set-policy.py           # Python скрипт для генерации файла с политиками сканирования
│   │   │   │   └── aisa-set-settings.py         # Python скрипт для генерации файла с настройками проекта
│   │   │   └── config                           # Директория со скриптами для конфигурации внутри Docker'а
│   │   │       └── install_packages.sh          # Bash скрипт для установки AISA клиента
│   │   └── Dockerfile                           # Пример конфигурации для docker образа под Linux
│   └── aisa-windows                             # Директория с примерами сборки docker образа под Windows 
│       ├── docker_build_data                    # Директория со скриптами для сборки Docker образа
│       │   ├── certs                            # Директория со скриптом
│       │   │   └── import-certs.ps1             # PowerShell скрипт для установки сертификатов
│       │   ├── create-json                      # Директория со скриптом
│       │   │   └── create-json.ps1              # PowerShell скрипт для генерации файла с настройками AISA
│       │   ├── download-and-unpack              # Директория со скриптом
│       │   │   └── download-and-unpack-auth.ps1 # PowerShell скрипт для загрузки и извлечения архива с авторизацией 
│       │   └── install-web                      # Директория со скриптом
│       │       └── install-web.ps1              # PowerShell скрипт для загрузки и установки бинарей с интернета
│       └── Dockerfile                           # Пример конфигурации для Docker образа под Windows
├── GitLab_templates                             # Директория с шаблонами для GitLab-CI
│   ├── default_project_files                    # Директория с примерами файлов с настройками проекта
│   │   ├── default_policy.json                  # 1 пример файла с настройками проекта
│   │   ├── default_project.aiproj               # 2 пример файла с настройками проекта
│   │   └── default_project_with_comments.aiproj # 3 пример файла с настройками проекта
│   ├── create_project.yml                       # 1 пример шаблона для GitLab CI
│   ├── example_1.yml                            # 2 пример шаблона для GitLab CI
│   ├── example_2.yml                            # 3 пример шаблона для GitLab CI
│   ├── existing_project.yml                     # 4 пример шаблона для GitLab CI
│   ├── non-existent_project.yml                 # 5 пример шаблона для GitLab CI
│   ├── no-wait-existing_project.yml             # 6 пример шаблона для GitLab CI
│   └── no-wait-non-existent_project.yml         # 7 пример шаблона для GitLab CI
├── TeamCity_meta-runners                        # Директория с метараннерами для TeamCity
│   ├── MetaRunnerAisaRunLinux.xml               # Пример метараннера для TeamCity под Linux
│   └── MetaRunnerAisaRunWindows.xml             # Пример метараннера для TeamCity под Windows
└── README.md                                    # Описание проекта и инструкции
```
