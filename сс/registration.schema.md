# CarrierCheck — блок-схема регистрации

Флоу регистрации в Chargebee-биллинге: форма → развилка по тарифу → чекаут → вебхуки → логика баннеров.

---

## 1. Регистрация: форма и развилка по тарифу

```mermaid
flowchart TD
  E["Шаг: email"]
  PC["Personal & company"]
  T["Шаг: tariff"]
  BD["Billing data"]
  S["Сабмит формы<br/><i>создаём юзера в API — vetting_count = 5, план = настоящий</i>"]
  MAIL["Письмо активации<br/><i>уходит всегда сразу, независимо от веток</i>"]
  Q{"Тариф?"}

  FREE["Прямые вызовы ЧБ<br/><i>create customer + subscription 0€</i>"]
  PAID["checkout_new_for_items<br/><i>редирект из формы на Hosted Checkout</i>"]
  ENT["Менеджер свяжется<br/><i>вышлет инвойс</i>"]

  FQ{"Вызовы ЧБ?"}
  FOK["customer + sub 0€ созданы"]
  FERR["ЧБ упал<br/><i>billing в кабинете пустой</i>"]
  FCONF["Confirmation (free)<br/><i>показывается всегда, даже если ЧБ упал</i>"]

  CHK["Юзер на чекауте"]
  WH["Webhook от ЧБ<br/><i>subscription_created</i>"]
  VC["Пополняем vetting_count<br/><i>до квоты оплаченного плана</i>"]
  DROP["Юзер остаётся в приложении<br/><i>бросил чекаут / offline — баннер, см. секцию 2</i>"]

  ECONF["Confirmation<br/><i>прямой путь, в ЧБ не ходим</i>"]

  E --> PC --> T --> BD --> S
  S --> MAIL
  S --> Q
  Q -->|"free"| FREE
  Q -->|"starter / business / business pro"| PAID
  Q -->|"enterprise"| ENT

  FREE --> FQ
  FQ -->|"ok"| FOK
  FQ -->|"ЧБ упал"| FERR
  FOK --> FCONF
  FERR --> FCONF

  PAID --> CHK
  CHK -->|"оплатил картой"| WH
  WH --> VC
  CHK -->|"бросил чекаут / offline"| DROP

  ENT --> ECONF

  class MAIL,WH,VC whBox
  class FREE,FOK,FCONF freeBox
  class PAID,CHK paidBox
  class ENT,ECONF entBox
  class FERR,DROP errBox
  classDef freeBox fill:#E1F5EE,stroke:#0F6E56,color:#04342C
  classDef paidBox fill:#E6F1FB,stroke:#185FA5,color:#042C53
  classDef entBox fill:#FAEEDA,stroke:#854F0B,color:#412402
  classDef whBox fill:#EAF3DE,stroke:#3B6D11,color:#173404
  classDef errBox fill:#FFFFFF,stroke:#5F5E5A,color:#2C2C2A
```

> Связка webhook ↔ юзер по **email** — нужно сделать email readonly в ЧБ.
> Доступ к проверкам гейтится квотой `vetting_count`, а не планом. Webhook при оплате пополняет квоту, план не трогает.

---

## 2. Определение баннера в приложении

Глобальный тост в хедере, на любой странице. Статус подписки тянется один раз за сессию.

```mermaid
flowchart TD
  Q1{"План free?"}
  Q2{"chargebee_customer_id есть?<br/><i>признак из модели юзера</i>"}
  Q3{"План enterprise?"}
  Q4{"Подписка active?<br/><i>статус из ЧБ</i>"}

  N0["Баннера нет"]
  N1["Баннера нет<br/><i>подписка active, оплачено</i>"]
  B1["Брошенный платный чекаут<br/><i>баннер «Вы не завершили оплату» → ссылка на /billing</i>"]
  B2["Enterprise ждёт инвойс<br/><i>баннер «активируется после оплаты счёта,<br/>менеджер свяжется» — без ссылки</i>"]
  B3["Offline payment — оплата не прошла<br/><i>баннер «ждём ваш платёж по инвойсу» → ссылка на /billing</i>"]

  Q1 -->|"да"| N0
  Q1 -->|"нет — платный / enterprise"| Q2
  Q2 -->|"нет"| Q3
  Q2 -->|"да"| Q4
  Q3 -->|"нет — брошен чекаут"| B1
  Q3 -->|"да"| B2
  Q4 -->|"active"| N1
  Q4 -->|"не active"| B3

  class B1 redBox
  class B2,B3 amberBox
  class N0,N1 grayBox
  classDef redBox fill:#FCEBEB,stroke:#A32D2D,color:#501313
  classDef amberBox fill:#FAEEDA,stroke:#854F0B,color:#412402
  classDef grayBox fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
```

### Как читается логика баннера

- Сначала проверяем `chargebee_customer_id` из модели юзера — если пусто, баннер определяется планом, запрос в ЧБ не нужен.
- Запрос статуса подписки в ЧБ — только когда `chargebee_customer_id` есть. Тянется один раз за сессию, хедер читает из стора.
- Доступность проверок (`vetting_count`) баннер не определяет — это отдельно решают webhook ЧБ и API.

| Состояние | Признак | Баннер |
|---|---|---|
| free | план free | нет |
| платный, нет `customer_id`, не enterprise | модель | «Вы не завершили оплату» → ссылка на /billing |
| enterprise, нет `customer_id` | модель | «активируется после оплаты счёта, менеджер свяжется» — без ссылки |
| есть `customer_id`, подписка active | ЧБ | нет |
| есть `customer_id`, подписка не active | ЧБ | «ждём ваш платёж по инвойсу» → ссылка на /billing |

---

## Ключевые решения

- **Billing data** в форме показывается только если тариф не free.
- **Письмо активации** уходит всегда сразу после сабмита, независимо от остальных веток.
- **Кастомер в API** создаётся на сабмите с настоящим планом и `vetting_count = 5`; доступ к проверкам гейтится квотой, а не планом.
- **free** — customer и подписка 0€ через прямые вызовы ЧБ. Если вызовы упали — юзер всё равно создан как free с 5 проверками, confirmation показывается, billing в кабинете пустой.
- **starter / business / business pro** — редирект из формы на Chargebee Hosted Checkout.
- **enterprise** — прямой путь к confirmation, в ЧБ не ходим; менеджер свяжется и вышлет инвойс.
- Webhook при оплате пополняет `vetting_count` до квоты плана. Связка webhook ↔ юзер по email — нужен readonly email в ЧБ.
