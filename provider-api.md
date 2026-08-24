# API для Провайдеров

## ⚡️ Быстрый старт (коллекция методов с автоматизацией подписи в Postman)

Для вашего удобства мы создали коллекцию в [postman](https://www.postman.com/), которая повторяет все методы API. Вы можете протестировать на ней функциональность и поиграть с запросами.
1. В postman выбрать пункт меню "Import file", например во вкладке "Home"
2. Импортировать [файл](providers.postman.json) в json формате
3. Перейти курсором на импортированную коллекцию, выбрать вкладку "Variables" и вписать вашу конфигурацию

---

## Схемы объектов

## TaskResource
- [StatusEnum](#statusenum)
```
{
  "id": "{{task_id}}",
  "order_id": "{{order_id}},
  "amount": "{{amount}}",
  "status": "{{StatusEnum}}",
  "reason": "{{reason}}", // nullable, only for returned tasks
  "card": "{{client card}}",
  "additional_fields": { // nullable
    "{{additional_field_key}}": "{{additional_field_value}}",
    ...
  },
  "created_at": "{{created_at}}", // ISO 8601 - YYYY-MM-DDTHH:mm:ssZ
}
```

### StatusEnum
| Статус                            | Описание                                         |
|:----------------------------------|:-------------------------------------------------|
| **new**                           | Новая задача на выплату                          |
| **returned-for-edit**             | Задача была возобновлена (оплата уже выполнена)  |
| **returned-for-processing**       | Задача была возобновлена (оплаты не было)        |

---

## Доступ к API 🔑

Для обеспечения безопасности используются 2 ключа: ключ авторизации и ключ подписи.
Оба ключа генерируются в ЛК на вкладке "API".

### API авторизация
Для авторизации в API протоколе используются ключ авторизации, который необходимо передать как Header:
```js
Authorization: "Bearer {{AuthKey}}"
```

### Подпись API запросов
В целях безопасности в запросах к API требуется подпись, которая хэширует тело запроса с помощью ключа подписи.
Сгенерированную подпись необходимо передать как Header:
```js
X-Signature: "{{Signature}}"
```

Сама подпись генерируется по алгоритму:
```php
hmac(sha256(SignatureKey, Request.JSON))
```
(Где SignatureKey — ключ подписи, а Request.JSON — json-тело конкретного отправляемого запроса).

Если json-тело запроса отсутствует, подпись делается от пустой строки.

---

## Заголовки для всех API запросов
```json
{
  "Accept": "application/json",
  "Authorization": "Bearer {{AUTH_KEY}}",
  "X-Signature": "{{GENERATED_SIGNATURE}}"
}
```

## Типовой ответ об ошибке для всех запросов:
```json
{
  "success": 0,
  "code": "{{4xx}}",
  "errors": {
    "message": "{{error_message}}",
    "details": {}
  }
}
```

---

## Методы API 🔠

### Проверить авторизацию (тестовый метод)

**[GET] /{{API_PROVIDER_PREFIX}}/**

#### Успешный ответ
```
{
  "success": 1,
  "code": 200,
  "data": "OK, you can use API"
}
```

### Получить список задач на выплату

**[GET] /{{API_PROVIDER_PREFIX}}/tasks/outbound**

#### Успешный ответ
[TaskResource](#taskresource)
```
{
  "success": 1,
  "code": 200,
  "data": [
    {{TaskResource}},
    ...
  ]
}
```

### Подтвердить задачу на выплату

**[POST] /{{API_PROVIDER_PREFIX}}/tasks/outbound/confirm**
```
{
  "id": "{{task_id}}",
  "transactions": [
    {
      "amount": "{{paid_amount}}",
      "receipt": "{{receipt}}" // file, allowed types: jpeg, png, pdf
    },
    ...
  ]
}
```

#### Успешный ответ
```
{
  "success": 1,
  "code": 200,
  "data": {
    "result": true
  }
}
```

### Отменить задачу на выплату

**[POST] /{{API_PROVIDER_PREFIX}}/tasks/outbound/cancel**
```
{
  "id": "{{task_id}}",
  "reason": "{{reason_for_cancellation}}" // max 50 characters
}
```

#### Успешный ответ
```
{
  "success": 1,
  "code": 200,
  "data": {
    "result": true
  }
}
```

----

## Webhook

При возникновении [события](#типы-событий), на URL-адрес, который установлен в поле `Webhook URL` в ЛК, производится отправка HTTP-запроса методом `POST`.

Если в ответе на запрос не был получен код ошибки `2xx`, будут производиться попытки переотправки с интервалами между каждой последующей попыткой в 180, 300, 600, 900, 1800, 3600 секунд.

#### Типы событий
| Статус                                | Описание                            |
|:--------------------------------------|:------------------------------------|
| **new_outbound_task**                 | Создана новая задача на выплату     |
| **returned_outbound_task**            | Задача на выплату была возобновлена |

#### Тело запроса
[TaskResource](#taskresource)
```
{
  "event": "new_outbound_task",
  "task": {{TaskResource}}
}
```
