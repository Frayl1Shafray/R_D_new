## LECTURE 14
**Завдання:** робота з MongoDB на тему спортзалу
**Ціль:** створення бази даних для спортзалу, в якій зберігатимуть дані про клієнтів, їхнє членство, тренування і тренерів. 
**Кроки для виконання:**
- Створення бази даних та колекцій:
    - Назвіть базу даних як gymDatabase
    Створення БД через compas

    ![alt text]({D2006D88-31FD-4FD2-B010-FC2FD9D14903}.png)
    ![alt text]({290C3A02-CA51-4E06-B83E-A3BAE56751AC}.png)

    - Створіть колекції: clients, memberships, workouts, trainers
    Створення колекцій через інтерфейс

    ![alt text]({4ABC7129-8681-46D6-80B5-254BFD4073BC}.png)

    ![alt text]({EE73C5BF-4E69-4CAD-9296-A05757B1776C}.png)
    Загальний вигляд:
    
    ![alt text]({98BE6BE5-1944-451D-AA54-70A953453D4E}.png)

- Визначення схеми документів:
    - Clients: client_id, name, age, email
    Створення схеми та заповнення колекцій даними

    ![alt text]({7CD2EC59-EA7D-4B6B-916C-0693D7A10E22}.png)


    Загальний вигля колекції


    ![alt text]({D3C02960-6690-49F2-8008-5EA056B7172A}.png)

    - Memberships: membership_id, client_id, start_date, end_date, type
    Заповення даними:
    ![alt text]({4DA64770-AE12-4B60-845C-F86B520B8D77}.png)
    
    - Workouts: workout_id, description, difficulty
    Аналогічне заповнення колецій
    ![alt text]({4A32EE59-0D09-4CE6-97D9-4142EBE0A4EC}.png)

    - Trainers: trainer_id, name, specialization

    ![alt text]({6D612F2E-9C40-4DEA-B9A0-2B87DC6F3738}.png)

- Заповнення колекцій даними:
    - Додайте кілька записів до кожної колекції

    Записи були одразу додані при побудові схеми

- Запити:
    - Знайдіть всіх клієнтів віком понад 30 років
    Запит через інтерфейс compas:
    код запиту в колекції client: `{ "age": { "$gt": 30 } }`
    ![alt text]({ED421274-053E-4611-BF32-1C6EAF2603EB}.png)

    - Перелічіть тренування із середньою складністю
    Код запиту: `{difficulty: "medium"}`
    ![alt text]({E5AC3388-34B1-496C-99CE-B89D87530CE8}.png)
    - Покажіть інформацію про членство клієнта з певним client_id
    Код запиту: { "client_id": "C002" }
    ![alt text]({3A4074D8-4479-46AC-BD6F-B47B1DE096C4}.png)

**Виконання запитів через консоль mongosh:**
- Знайдіть всіх клієнтів віком понад 30 років
    ```js
    use gymDatabase
    switched to db gymDatabase
    db.clients.find({age: {$gt:30}})
    ```
    ![alt text]({A393BD16-0AE2-44E3-BF7C-6D4640D9CD47}.png)
- Тренування із сердньою скаладністю:
    ```js
    db.workouts.find({difficulty: "medium"})
    ```
    ![alt text]({D50D1955-1DF4-4228-BF18-75F6EC261FBF}.png)

- Покажіть інформацію про членство клієнта з певним client_id
    ```js
    db.memberships.find({client_id: "C003"})
    ```
    ![alt text]({E373FB56-79C1-4513-9BF3-5C1E832FC3A7}.png)
