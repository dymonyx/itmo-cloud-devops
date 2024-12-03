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
      uses: actions/checkout@v3

    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x' 

    - name: Restore dependencies
      run: dotnet restore sem_1/test

    - name: Build
      run: dotnet build --no-restore sem_1/test

    - name: Run static analysis
      run: dotnet format sem_1/test/testapp/testapp.csproj analyzers

    - name: Publish
      run: dotnet publish sem_1/test/testapp/testapp.csproj -c Release -o output

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'

    - name: Deploy application
      run: echo "deploy was successful"

```
## "Хороший" CI/CD
`
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

jobs:
  build:
    timeout-minutes: 20
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        os: [ubuntu-24.04, windows-2022]

    steps:
    - name: Checkout code
      uses: actions/checkout@eef61447b9ff4aafe5dcd4e0bbf5d482be7e7871

    - name: Setup .NET
      uses: actions/setup-dotnet@6bd8b7f7774af54e05809fcc5431931b3eb1ddee
      with:
        dotnet-version: ${{ vars.DOTNET_VERSION }}
        
    - name: Cache .NET dependencies
      uses: actions/cache@6849a6489940f00c2f30c0fb92c6274307ccb58a
      with:
        path: ${{ github.workspace }}\.nuget\packages
        key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj', '**/*.sln') }}
        restore-keys: |
          ${{ runner.os }}-nuget-

    - name: Restore dependencies
      run: dotnet restore sem_1/test

    - name: Build
      run: dotnet build --no-restore sem_1/test

    - name: Run static analysis
      run: dotnet format sem_1/test/testapp/testapp.csproj analyzers

    - name: Publish
      run: dotnet publish sem_1/test/testapp/testapp.csproj -c Release -o output 
      
    - name: Upload build artifacts
      uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882
      with:
        name: build-output-${{ matrix.os }}
        path: output/

  deploy:
    needs: build
    runs-on: ubuntu-24.04
    if: github.ref == 'refs/heads/main'

    steps:
    - name: Checkout code
      uses: actions/checkout@eef61447b9ff4aafe5dcd4e0bbf5d482be7e7871
      
    - name: Download build artifacts
      uses: actions/download-artifact@fa0a91b85d4f404e444e00e005971372dc801d16
      with:
        name: build-output-ubuntu-24.04

    - name: Setup .NET
      uses: actions/setup-dotnet@6bd8b7f7774af54e05809fcc5431931b3eb1ddee
      with:
        dotnet-version: ${{ vars.DOTNET_VERSION }}

    - name: Deploy application
      run: echo "deploy was successful"
        
        

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
      uses: actions/checkout@v3

    - name: Setup .NET
      uses: actions/setup-dotnet@v3
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

### Кэширование и artifacts
    - name: Cache .NET dependencies
      uses: actions/cache@6849a6489940f00c2f30c0fb92c6274307ccb58a
      with:
        path: ${{ github.workspace }}\.nuget\packages
        key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj', '**/*.sln') }}
        restore-keys: |
          ${{ runner.os }}-nuget-

    - name: Upload build artifacts

    - name: Download build artifacts
Использование кэширования ускоряет сборку, так как при повторном запуске зависимости будут браться из кэша, а не устанавливаться заново. `Artifacts` позволяют сохранить результаты сборки между `job`'ами, чтобы не выполнять одно и то же несколько раз.
## Запуск Github Actions
[репа с проектом](https://github.com/dymonyx/csharp_course)
### 'Плохой' CI/CD
![](img/Pasted%20image%2020241203191628.png)
### 'Хороший' CI/CD
![](img/Pasted%20image%2020241203191717.png)
## Рефлексия
![](img/Pasted%20image%2020241019163012.png)

тяжило нипонятно и как куда что =/..
## Sources
[gha best practices](https://exercism.org/docs/building/github/gha-best-practices)
[gha security practices](https://www.stepsecurity.io/blog/github-actions-security-a-case-study-with-google)
