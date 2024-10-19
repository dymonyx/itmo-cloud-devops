## Задачи
1. Написать “плохой” `CI/CD` файл, который работает, но в нем есть не менее пяти `bad practices` по написанию `CI/CD`
2. Написать “хороший” `CI/CD`, в котором эти плохие практики исправлены
3. В Readme описать каждую из плохих практик в плохом файле, почему она плохая и как в хорошем она была исправлена, как исправление повлияло на результат
4. Прочитать историю про Васю (она быстрая, забавная и того стоит): [https://habr.com/ru/articles/689234/](https://habr.com/ru/articles/689234/)
## "Плохой" `CI/CD`
Пока что у меня нет никаких проектов, в которых нужно было бы разворачивать `CI/CD` пайплайн, но в следующем семестре мы будем писать приложение на `C#`, поэтому я напишу `CI/CD` файл для предстоящего проекта с помощью `Github Actions`. Если честно, бОльшая часть найденной инфы про `bad practices` касается именно эксплуатации `CI/CD`, а не написания ямликов, поэтому было тяжило, но что-то вроде нашлось..
![](img/Pasted%20image%2020241019133437.png)

Представляю вашему вниманию ямлик с "плохими практиками":
```
name: bad workflow

on: push

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout

    - name: Setup .NET
      uses: actions/setup-dotnet
      with:
        dotnet-version: '8.0.x' 

    - name: Restore dependencies
      run: dotnet restore

    - name: Build
      run: dotnet build --no-restore

    - name: Test
      run: dotnet test --no-build

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
    - name: Checkout code
      uses: actions/checkout

    - name: Setup .NET
      uses: actions/setup-dotnet
      with:
        dotnet-version: '8.0.x'

    - name: Publish
      run: dotnet publish -c Release -o output

```
## "Хороший" `CI/CD`
Сразу покажу и исправленный ямлик:
```
name: good workflow

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  DOTNET_VERSION: '8.0.x'

jobs:
  build:
    timeout-minutes: 15
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        os: [ubuntu-24.04, windows-2022]
        dotnet: ${{ env.DOTNET_VERSION }}

    steps:
    - name: Checkout code
      uses: actions/checkout@eef61447b9ff4aafe5dcd4e0bbf5d482be7e7871

    - name: Setup .NET
      uses: actions/setup-dotnet@6bd8b7f7774af54e05809fcc5431931b3eb1ddee
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}

    - name: Restore dependencies
      run: dotnet restore

    - name: Build
      run: dotnet build --no-restore

    - name: Test
      run: dotnet test --no-build  

  deploy:
    needs: build
    runs-on: ubuntu-24.04
    if: github.ref == 'refs/heads/main'

    steps:
    - name: Checkout code
      uses: actions/checkout@eef61447b9ff4aafe5dcd4e0bbf5d482be7e7871

    - name: Setup .NET
      uses: actions/setup-dotnet@6bd8b7f7774af54e05809fcc5431931b3eb1ddee
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}

    - name: Publish
      run: dotnet publish -c Release -o output
        

```
Итак, в чём отличия..?
![](img/Pasted%20image%2020241019150543.png)
### triggers
```
on: push
```
сменилось на
```
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
```
Нет никакого смысла запускать тесты и сборку при каждом малейшем пуше в любую ветку и использовать ограниченные ресурсы *(не забываем, что у нас `free tier`)* на какие-то неокончательные идеи (раз они в `main` не идут). А ещё среди логов из всех веток будет труднее найти возникшую ошибку, чем только из `main`. *+воркфлоу отрабатывает теперь ещё и на `pull_request`*
### concurrency
```
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```
Добавление этого блока позволяет отменить предыдущей экземпляр воркфлоу, если уже запущен новый. Это позволяет экономить ресурсы и избегать конфликтов. *(в логах, опять же, будет легче разобраться)*
### ENV
```
env:
  DOTNET_VERSION: '8.0.x'
```
Объявление переменной окружения, описывающей версию `dotnet` делает ямлик понятнее и читабельней, а также в дальнейшем при изменении версии, её придётся поменять только в одном месте, что сэкономит немного времени.

### timeout
```
timeout-minutes: 15
```
Обратимся к документации гх:
>**Job execution time** - Each job in a workflow can run for up to 6 hours of execution time. If a job reaches this limit, the job is terminated and fails to complete.

Если в коде допущена ошибка, заставляющая джобу `build` выполняться бесконечное количество времени, то только спустя 6ч гх её прервёт. Чтобы не ждать столько времени и быстрее исправить ошибку, можно установить лимит на выполнение джобы. Также зависшие воркфлоу нам не нужны, потому что они потербляют ресурсы без надобности.
### matrix strategy + stable version
```
runs-on: ubuntu-latest
```
сменилось на
```
strategy:
  matrix:
	os: [ubuntu-24.04, windows-2022]
```

Теперь проект будет тестироваться на нескольих осях, что позволит ему поддерживать кроссплатформенность, а точное указание версий предотвращает появление дополнительных ошибок, связанных с выходом новых версий (т.к `latest` время от времени начинает указывать на другую версию оси).

### pin actions (SHA)
```
steps:
    - name: Checkout code
      uses: actions/checkout

    - name: Setup .NET
      uses: actions/setup-dotnet
```
сменилось на 
```
steps:
    - name: Checkout code
      uses: actions/checkout@eef61447b9ff4aafe5dcd4e0bbf5d482be7e7871

    - name: Setup .NET
      uses: actions/setup-dotnet@6bd8b7f7774af54e05809fcc5431931b3eb1ddee
```
Тут мы фиксируем версии `actions` на определённых комитах, чтобы сделать билд стабильнее и, снова же, избежать непредвиденных ошибок, связанных с обновлением `actions`.
## Рефлексия
![](img/Pasted%20image%2020241019163012.png)

тяжило нипонятно и как куда что =/..
## Sources
[gha best practices](https://exercism.org/docs/building/github/gha-best-practices)
[gha security practices](https://www.stepsecurity.io/blog/github-actions-security-a-case-study-with-google)
