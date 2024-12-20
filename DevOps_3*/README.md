## Задачи
1. Сделать красиво работу с секретами. Например, поднять `Hashicorp Vault` (или другую секретохранилку) и сделать так, чтобы `CI/CD` пайплайн (или любой другой ваш сервис) ходил туда, брал секрет, использовал его не светя в логах. 
2. В `Readme` аргументировать почему ваш способ красивый, а также описать, почему хранение секретов в `CI/CD` переменных репозитория не является хорошей практикой.

[репа с проектом](https://github.com/dymonyx/csharp_course), [лаба про CI/CD](https://github.com/dymonyx/itmo-cloud-devops/tree/main/DevOps_3)
## Выбор хранилки
Вот [тут](https://thectoclub.com/tools/best-secrets-management-tools/#full-listing-2512886) описаны **10 Best Secrets Management Tools of 2024**, выбор пал на `Bitwarden`. Был зареган аккаунт, создан проект и так как в моём проекте не используются секреты, то в качестве примера секретом будет выступать единственная переменная окружения `DOTNET_VERSION` (которая до этого хранилась в переменных окружения гитхаба):
![](img/Pasted%20image%2020241212013126.png)
## CI/CD

У `Bitwarden` есть [статья](https://bitwarden.com/blog/using-bitwarden-secrets-manager-and-github-actions/) на тему коннекта их хранилки с `gh actions`, а ещё вспомогательная вот [эта](https://bitwarden.com/help/access-tokens/) про токены ([линк](https://github.com/bitwarden/sm-action) на их `gh action` для секретов).

В соответствии с их инструкциями был добавлен новый шаг в `CI/CD` (и чуть переписано использование переменной `DOTNET_VERSION`, ибо берётся она теперь из другого места):
```
steps:
- name: Bitwarden Get Secrets
  uses: bitwarden/sm-action@92d1d6a4f26a89a8191c83ab531a53544578f182
  with:
    access_token: ${{ secrets.BITWARDEN_TOKEN }}
    secrets: |
	  3c8a49b0-1c41-4950-a6e1-b24301728d13 > DOTNET_VERSION
```

```
dotnet-version: ${{ env.DOTNET_VERSION }}
```
Запустим пайплайн и проверим логи:
![](img/Pasted%20image%2020241212032225.png)

Версия `dotnet` не светится и успешно используется пайплайном.
Поменяем версию в секрете с `8.0.x` на `7.0.x`:
![](img/Pasted%20image%2020241212033412.png)

Убеждаемся, что секрет успешно применяется, потому что пайплан полетел)) проект не заточен под эту версию.
![](img/Pasted%20image%2020241212033552.png)

## Аргументация
- **Why способ красив?** Он быстрый в реализации и интуитивно понятный - нужно просто зарегаться на сайте, добавить секрет, сформировать токен доступа, скопировать айди секрета и по шаблону из документации добавить один небольшой шаг в джобу/-ы.
- **Why хранение секретов в переменных репозитория не очень?** Во-первых, это не очень безопасно, например, если доступ к репозиторию получит малыш-стажёр, он может просто натыкать не туда, куда надо.. Во-вторых, в любой секретохранилке фич всегда будет больше (можно ограничить круг лиц, кому доступны секреты, посмотреть логи использования секретов), всё же они заточены именно под работу с секретами. В-третьих, хранилку можно подружить с кучей разных инструментов (гитлаб, кубер, ансибл).