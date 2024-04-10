# AppSecCloudCamp
Тестовое задание на стажировку AppSecCloudCamp
## 1. Вопросы для разогрева
1. К сожалению, ещё не приходилось
2. К сожалению, ещё не приходилось
3. В рамках CTF-соревнований, а также поднимая  bWAPP неоднократно приходилось искать уязвимости в web-прилояжениях, используя BurpSuite
4. Я студент, обучающийся по направлению “Информационная безопасность”.  Несколько лет назад мне довелось участвовать в соревнованиях по системному администрированию + используя методологией DevOps. После, я узнал, что можно совместить то, чем мне нравится заниматься вне учебного времени со специальностью, на которой я учусь и которую довольно неплохо освоил (даже работал какое-то время в Ростелеком-Солар и сопровождал СЗИ)-DevSecOps. И считаю, что данная стажировка очень подоходит для развития моих желаемых навыков в этой области.
## **2. Security code review**
### **Часть 1. Security code review: GO**
Исходный код:
```
package main

import (
    "database/sql"
    "fmt"
    "log"
    "net/http"
    "github.com/go-sql-driver/mysql"
)

var db *sql.DB
var err error

func initDB() {
    db, err = sql.Open("mysql", "user:password@/dbname")
    if err != nil {
        log.Fatal(err)
    }

err = db.Ping()
if err != nil {
    log.Fatal(err)
    }
}

func searchHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != "GET" {
        http.Error(w, "Method is not supported.", http.StatusNotFound)
        return
    }

searchQuery := r.URL.Query().Get("query")
if searchQuery == "" {
    http.Error(w, "Query parameter is missing", http.StatusBadRequest)
    return
}

query := fmt.Sprintf("SELECT * FROM products WHERE name LIKE '%%%s%%'", searchQuery)
rows, err := db.Query(query)
if err != nil {
    http.Error(w, "Query failed", http.StatusInternalServerError)
    log.Println(err)
    return
}
defer rows.Close()

var products []string
for rows.Next() {
    var name string
    err := rows.Scan(&name)
    if err != nil {
        log.Fatal(err)
    }
    products = append(products, name)
}

fmt.Fprintf(w, "Found products: %v\n", products)
}

func main() {
    initDB()
    defer db.Close()

http.HandleFunc("/search", searchHandler)
fmt.Println("Server is running")
log.Fatal(http.ListenAndServe(":8080", nil))
}
```
### Уязвимсоти:
1. Хранение учётных данных в явном виде в коде:
```
db, err = sql.Open("mysql", "user:password@/dbname")
```
Данная уязвимость может привести к утечке учётных данных для доступа к базе данных

**Исправление:** хранить учёные данные в переменных окружения

2. SQL-injection
```
query := fmt.Sprintf("SELECT * FROM products WHERE name LIKE '%%%s%%'", searchQuery)
http.HandleFunc("/search", searchHandler)

```
Данная уязвимость может привести как к краже учётных данных пользователей и прочей персональной информации о них, так и, если БД запущена от root, можно закинуть revers shell что в дальнейшем может привести к полному взлому системы

**Исправление:** Использовать подготовленные запросы

Помимо этого: экранирование, использование целочисленных значений 

Использование подготовленных запросов, на мой взгляд, является лучшим методом защиты, т.к. переменные отправляются отдельно от запроса и не могут влиять на него, нет неоходимости их эранировать, сервер использует эти значения непосредственно в момент выполнения, уже после того, как был обработан шаблон выражения

### **Часть 2: Security code review: Python**

**Пример №2.1**
Исходный код:
```
from flask import Flask, request
from jinja2 import Template

app = Flask(name)

@app.route("/page")
def page():
    name = request.values.get('name')
    age = request.values.get('age', 'unknown')
    output = Template('Hello ' + name + '! Your age is ' + age + '.').render()
return output

if name == "main":
    app.run(debug=True)
```

**Server-side Template Injection-уязвимость**
```
output = Template('Hello ' + name + '! Your age is ' + age + '.').render()
```
Эксплуатация данной уязвимости может привести к внедрению вредоносного кода в шаблон, вплоть до reverse-shell, что может служить начальной точкой для попадания злоумышленника внуть системы

**Исправление:** Использование шаблонов без логики

**Пример №2.2**
Исходный код:
```
from flask import Flask, request
import subprocess

app = Flask(name)

@app.route("/dns")
def dns_lookup():
    hostname = request.values.get('hostname')
    cmd = 'nslookup ' + hostname
    output = subprocess.check_output(cmd, shell=True, text=True)
return output
if name == "main":
    app.run(debug=True)
```
**Command Injection**
```
cmd = 'nslookup ' + hostname
output = subprocess.check_output(cmd, shell=True, text=True)
```
Данная уязвимость может привести к выполнению системных команд на сервере, в результате чего злоумышленник может навредить системе, прокинув reverse-shell для дальнейшего продвижения в системе и получения прав администратора на целевой системе

**Исправление:** Валидация пользовательского ввода, использование whitelist’ов, использование регулярных выражений

Лучшим способом будет просто запретить пользователям использовать команды, тогда эта уязвимость просто сойдёт на нет, но если в этом всё же есть необходимость, то лучше  использовать регулярные выражения

## **Часть 3. Моделировани угроз**

Проанализируйте диаграмму потоков данных приложения и ответьте на следующий вопросы:

- Расскажите, какие потенциальные проблемы безопасности существуют для данного сервиса?
- Расскажите, к каким последствиям может привести эксплуатация проблем, найденных вами?
- Расскажите, какие способы исправления уязвимостей и смягчения рисков вы можете предложить по отмеченным вами проблемам безопасности?
- Напишите список уточняющих вопросов, которые вы бы задали разработчикам данного сервиса?


  **Ответ:**
1. Расскажите, какие потенциальные проблемы безопасности существуют для данного сервиса?
    1. SQL-инъекции
    2. SSRF
    3. CSRF
    4. XSS
    5. Проблемы с авторизацией
    6. Уязвимые библиотеки в частях микрофронта
  
2. Расскажите, к каким последствиям может привести эксплуатация проблем, найденных вами?
    1. Компрометация базы данных, включая информацию о пользователях, и кражу учётных данных пользователей с повышеными привилегиями, что может привести к полному контролю злоумышленника над системой
    2. Вплоть до компрометации данных 
    3. Компрометация сессии другого пользователя, выполнение различных команд от лица другого пользователя
    4. Кража сессии и данных другого пользователя 
    5. Возможность создать пользователя с повышенными правами, утечка данных, кража сесси другого пользователя 
    6. Отказ в обслуживании, RCE
  
3. Расскажите, какие способы исправления уязвимостей и смягчения рисков вы можете предложить по отмеченным вами проблемам безопасности?
    1. Использование подготовленных выражений, экранирование значений, 
    2. Проверка вводимых пользователем данных, использование whitelist’ов с предопределёнными URL’ми
    3. synchronizer token / double submit cookies
    4. Валидация введённых данных
    5. Многофакторная аутентификация
    6. Использование SCA

4. Напишите список уточняющих вопросов, которые вы бы задали разработчикам данного сервиса?
    1. Стоит ли у вас WAF, SIEM и прочие средства безопасности ?
    2. Есть ли у явное разграниченная DMZ ?
    3. Как проходит тестирование кода ? Какие средства автоматического тестирования используются ?
    4. Регуялярно ли, и, если руглярно, то как часто, проводятся аудиты безопасности ?














