# Лабораторна робота 1. Робота з СУБД PostgreSQL та основи SQL

## Загальна інформація

**Здобувач освіти:** Стрижеус Софія Андріївна
**Група:** ІПЗ-32
**Обраний рівень складності:** 2

## Виконання завдань

### Список таблиць

```sql
-- Запит для отримання списку таблиць
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;
```

Результат: У базі даних створено 8 основних таблиць: categories, customers, employees, order_items, orders, products, regions, suppliers.


<img width="450" height="745" alt="image" src="https://github.com/user-attachments/assets/502f7e5f-910d-435e-8b33-3cfa950bce6c" />

...


### Отримати всі записи з таблиці customers.

```sql
SELECT * FROM customers;
```

Результат: Отримано 15 записів клієнтів, включаючи як фізичних осіб, так і юридичні особи з різних міст України.

<img width="1004" height="304" alt="image" src="https://github.com/user-attachments/assets/3524a639-fd2d-4b8f-bb4b-f2d1a8f64581" />
<img width="1004" height="97" alt="image" src="https://github.com/user-attachments/assets/29f87437-5682-45a8-9266-bc12faf79c14" />
<img width="1004" height="97" alt="image" src="https://github.com/user-attachments/assets/074f9662-3757-4726-8166-fdffd8b2f160" />
Рівень 1. Основні SELECT запити

1.1 Ввести тільки назви товарів їх ціни з таблиці товарів

<img width="361" height="42" alt="image" src="https://github.com/user-attachments/assets/548bf898-729a-4b3b-b733-f7626585eeab" />


Результат: виведено 25 найменувань товарів та їх вартість у гривнях

<img width="735" height="765" alt="image" src="https://github.com/user-attachments/assets/1ba3e4bd-ff4b-4f80-b913-16d812615030" />
<img width="788" height="689" alt="image" src="https://github.com/user-attachments/assets/b37686b5-acf2-4a76-a588-09f7e9731bcc" />
<img width="788" height="51" alt="image" src="https://github.com/user-attachments/assets/f623ac6d-c659-44c8-a2aa-bfb0b9ca6a88" />

1.2 Показати контактні дані всіх співробітників

<img width="604" height="65" alt="image" src="https://github.com/user-attachments/assets/be40a940-c0af-47d5-80a6-be5fbea3cdec" />

Результат: отримано інформація по всіх співробітниках

<img width="850" height="569" alt="image" src="https://github.com/user-attachments/assets/c80ac852-7405-4d2b-b6f7-e523636f9812" />

2. Прості умови WHERE

2.1 Знайти всіх клієнтів з міста Київ

<img width="510" height="82" alt="image" src="https://github.com/user-attachments/assets/d1d4d18e-be23-4ead-aba6-81af05eb7206" />

Результат: знайдено 4 клієнтів зареєстованих у місті Київ

<img width="925" height="280" alt="image" src="https://github.com/user-attachments/assets/dd3d165e-fbc7-45d2-85c9-69e634b606eb" />

2.2 Вивести товари, які коштують більше 25000 грн

<img width="341" height="92" alt="image" src="https://github.com/user-attachments/assets/fa8a813d-93a9-4eb5-b516-3a50d5145ecf" />

Результат: відібрано 13 позицій товарів преміум-сегменту

<img width="483" height="578" alt="image" src="https://github.com/user-attachments/assets/f2ad6cf5-d7b7-45f1-9aab-722c40afc66f" />
<img width="491" height="85" alt="image" src="https://github.com/user-attachments/assets/c4a256bf-9fcb-4812-8b77-f4ff6d3d0c1f" />

2.3 Показати всі замовлення зі статусом 'delivered'

<img width="640" height="77" alt="image" src="https://github.com/user-attachments/assets/1fe17e83-3d47-490a-8199-0f4e3b286ffe" />

Результат: отримано 26 успішно доставлених замовлень

<img width="627" height="585" alt="image" src="https://github.com/user-attachments/assets/902227cf-65a0-48cf-907f-3064f0e37044" />
<img width="627" height="483" alt="image" src="https://github.com/user-attachments/assets/917ae3dd-942e-4bbb-8719-49e52ff07336" />
<img width="627" height="172" alt="image" src="https://github.com/user-attachments/assets/84c10529-3343-4b07-b259-e14271ad1f47" />

2.4 Знайти співробітників які працюють у відділі продажів

<img width="437" height="80" alt="image" src="https://github.com/user-attachments/assets/a88a8dfe-97e9-4ff5-a4a8-763720dc7ee2" />

Результат: знайдено 3 менеджерів з продажу у Києві, Львові та Харкові

<img width="570" height="226" alt="image" src="https://github.com/user-attachments/assets/64365b0b-ffe1-4053-bdfd-0a3142997840" />

3. Базове сортування ORDER BY

3.1 Відсортувати товари за зростанням ціни

<img width="347" height="82" alt="image" src="https://github.com/user-attachments/assets/f636b3c0-1f30-40d1-bd4b-21552a4fcf95" />

Результат: товари впорядковано від найдешевшого

<img width="596" height="531" alt="image" src="https://github.com/user-attachments/assets/98460568-f3f6-4f58-8592-c3d8aaca086d" />
<img width="602" height="532" alt="image" src="https://github.com/user-attachments/assets/5fefa016-914d-4b93-acff-0df08dbfb36d" />
<img width="662" height="130" alt="image" src="https://github.com/user-attachments/assets/9f1df290-b5bb-4fef-b87b-8c5189737d71" />

3.2 Показати клієнтів в алфовітному порядку за іменем контактної особи

<img width="362" height="77" alt="image" src="https://github.com/user-attachments/assets/177d4fd2-aded-4bc0-b283-2b724ebc2391" />

Результат: виведено 15 клієнтів відсортованих за алфовітом

<img width="640" height="530" alt="image" src="https://github.com/user-attachments/assets/48bc7e0c-b4a9-4df3-9c75-b41577a77885" />
<img width="631" height="131" alt="image" src="https://github.com/user-attachments/assets/9ee37597-218e-40c2-aa23-64845584fa4d" />

3.3 Вивести замовлення від найновіших до найстаріших

<img width="592" height="87" alt="image" src="https://github.com/user-attachments/assets/58b85567-acbc-4e06-b170-070e0c764891" />

Результат: замовлення відсортовано від 20.08.2024 до 15.01.2024

<img width="496" height="523" alt="image" src="https://github.com/user-attachments/assets/67a5593e-f15e-4fbb-9497-9a9e4e923b7f" />
<img width="505" height="490" alt="image" src="https://github.com/user-attachments/assets/e377b069-c7a3-40bc-9e02-53b7d4f280ae" />
<img width="510" height="397" alt="image" src="https://github.com/user-attachments/assets/d68a0086-45a9-4030-a851-37652c019846" />

4. Обмеження результатів LIMIT

4.1 Показати перші 10 найдорожчих товарів

<img width="398" height="105" alt="image" src="https://github.com/user-attachments/assets/e0faba8a-a6c0-46c4-ad76-3b866f488e6f" />

Результат: отримано топ-10 найдорожчих товарів магазину

<img width="507" height="498" alt="image" src="https://github.com/user-attachments/assets/b0f1d3a3-93d8-49ae-8add-fe7bcc0c9c88" />

4.2 Вивести 5 останніх замовлень

<img width="592" height="102" alt="image" src="https://github.com/user-attachments/assets/9712c2d8-31c0-4b50-962a-762cc7f2b745" />

Результат: отримано 5 найсвжіших замовлень

<img width="538" height="331" alt="image" src="https://github.com/user-attachments/assets/00bac73a-a258-4156-b2b7-cf17b6268d6d" />

4.3 Отримати перших 8 клієнтів в алфавітному порядку

<img width="445" height="102" alt="image" src="https://github.com/user-attachments/assets/583e8a5b-364e-4f24-96ce-5e4fbe9529e7" />

Результат: отримано перші 8 записів клієнтів з відсортованого за алфавітом списку

<img width="572" height="408" alt="image" src="https://github.com/user-attachments/assets/fe22226a-2bb9-4cbb-8804-fc2bf93dbcec" />

Рівень 2. Пошук за зразком з LIKE

1.1 Знайти всіх клієнтів чиї імена починаються на "Іван"

<img width="718" height="82" alt="image" src="https://github.com/user-attachments/assets/01996b19-6afe-40b2-b172-73910760a423" />

Результат: знайдено клієнта

<img width="640" height="100" alt="image" src="https://github.com/user-attachments/assets/6626ba40-9354-4388-8471-aae2fa182dfa" />

1.2 Вивести ровари у назві або описі яких є слово "телефон"

<img width="690" height="85" alt="image" src="https://github.com/user-attachments/assets/add1fe5b-878e-47ff-ae4f-cacb9d2718ef" />

Результат: знайдено записи у яких згадується слово "телефон"

<img width="710" height="108" alt="image" src="https://github.com/user-attachments/assets/67e213b3-bdfc-4c69-a201-fff7f4b0d738" />

1.3 Самостійні запити з LIKE

<img width="860" height="387" alt="image" src="https://github.com/user-attachments/assets/21282a10-6e4d-43c2-9184-8db56301a22d" />

Результат: отримано перелік клієнтів з поштою та співробітників з відповідними прізвищами

<img width="485" height="311" alt="image" src="https://github.com/user-attachments/assets/1bf7f73b-babd-4caa-b73d-453b978779f0" />

2. Логічні оператори AND, OR, NOT

2.1 Знайти товари дорожчі за 15000 грн і дешевші за 50000 грн

<img width="497" height="93" alt="image" src="https://github.com/user-attachments/assets/a41cd496-eeff-4d1f-979e-96c000a29660" />

Результат: відібрано 15 товарів 

<img width="492" height="527" alt="image" src="https://github.com/user-attachments/assets/b42ee577-41aa-40ff-a662-3927414cd37e" />
<img width="492" height="228" alt="image" src="https://github.com/user-attachments/assets/8f814e06-2a07-45a1-9063-261c2d20443d" />

2.2 Вивести клієнтів з Києва або Львова, які є юридичними особами

<img width="691" height="80" alt="image" src="https://github.com/user-attachments/assets/ea275d2e-6261-412f-8de9-100a98ae4f4e" />

Результат: знайдено 3 корпоративних клієнтів у місті Київ

<img width="612" height="177" alt="image" src="https://github.com/user-attachments/assets/e6dfba1c-5483-40b1-8260-e402bf672a6d" />

2.3 Самостійні запити з логічними операторами

<img width="862" height="522" alt="image" src="https://github.com/user-attachments/assets/c199c6ef-422d-4cce-a6ba-bc29069f12ba" />

Результат: отримано аналітичні вибірки для відділу закупівель, логістики та HR

<img width="562" height="136" alt="image" src="https://github.com/user-attachments/assets/b58169c3-1f2b-4f25-a9a3-a063fbc71a5a" />

3. Оператори IN, BETWEEN, NULL

3.1 Вивести клієнтів з міст Київ, Харків, Одеса, Дніпро

<img width="525" height="82" alt="image" src="https://github.com/user-attachments/assets/c07eeb55-ec0a-484a-a2e0-83abc1f146ec" />

Результат: отримано 12 клієнтів із зазначених міст-мільйонників

<img width="572" height="312" alt="image" src="https://github.com/user-attachments/assets/d521c39a-3278-4750-98b7-86d752f13fed" />
<img width="583" height="267" alt="image" src="https://github.com/user-attachments/assets/80befe73-0a48-44aa-be67-32cb5c53aea9" />

3.2 Знайти товаи в ціновій категорії від 10000 до 30000 грн

<img width="457" height="86" alt="image" src="https://github.com/user-attachments/assets/4e60bf99-e140-4afa-a250-b53e3cd07f2d" />

Результат: знайдено 13 товарів у вазаному діапазоні

<img width="502" height="480" alt="image" src="https://github.com/user-attachments/assets/d3362662-9eb4-427f-bf82-bfb5a9a7b9ff" />
<img width="502" height="175" alt="image" src="https://github.com/user-attachments/assets/c05fd772-f3a8-4f79-81ec-b10e73bae60c" />


3.3 Самостійні запити з IN, BETWEEN, IS NULL/IS NOT NULL

<img width="588" height="402" alt="image" src="https://github.com/user-attachments/assets/a095a04e-4185-47dc-90c6-849d49dea8ab" />

Результат: усі 6 запитів успішно відпрацювали та повернули очікувані набори запитів

<img width="772" height="308" alt="image" src="https://github.com/user-attachments/assets/af650cb7-a80f-4087-a757-c92383690bd3" />

4. Комбінування умов

<img width="835" height="478" alt="image" src="https://github.com/user-attachments/assets/073ee3ab-5b9b-42e7-a9eb-14f84dc0b90e" />
<img width="385" height="98" alt="image" src="https://github.com/user-attachments/assets/acbb0cb6-849d-4768-9520-8ef8b49abf93" />

Результат: складні комбіновані фільтри коректно поєднали логічні оператори перевірку діапазонів та шаблонів

<img width="780" height="315" alt="image" src="https://github.com/user-attachments/assets/84189854-58fe-4d70-b0c2-b20ad3a0e52b" />

5.Складне сортування та пагінація

<img width="607" height="403" alt="image" src="https://github.com/user-attachments/assets/e1f57f1b-f3a2-4d82-824f-3237ab6e069f" />

Результат: перевірено багаторівневе сортування за кількома стовпцями та посторінкове виведення результатів за допомогою LIMIT та OFFSET

<img width="385" height="492" alt="image" src="https://github.com/user-attachments/assets/3b1239f1-4eb0-4912-b6ac-6bc36c304e90" />

## Висновок
Під час виконання лабораторної роботи №1 було освоєно основи роботи з хмарною реляційною СУБД PostgreSQL на платформі Supabase. Успішно імпортовано навчальну базу даних інтернет магазину "ТехноМарт", досліжєено її структуру та зв'язкки між таблицчми. На практиці відпрацьовано написання SQL-запитів використання операторів вибірки, фільтрації, логічих операторів, сортування та пагінації.

**Самооцінка**: 4

**Обгрунтування**: У павному обсязі виконано всі обов'язкові та самостійні завдання 1-го та 2-го рівнів склладності
