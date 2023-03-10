## Сущности сервиса (Глоссарий)

{{% section %}}

    {{% note %}}
        Сначала обобщить все, и потом пойти по частностям. 
    {{% /note %}}

- Лицензия
- Провайдер выдачи лицензии 
- Фильтр
- Правило
- Заявка на выдачу/отзыв

---

### Лицензия

    {{% note %}}
Лицензия - это сущность описывает лицензию, параметры выдачи (провайдер и ключ выдачи), финансовые параметры лицензии, как и кому она выдается(сублицензирование или лицензирование только на ЮЛ) и прочее. Может выдаваться как автоматически, так и вручную в зависимости от параметров самой лицензии.   
    {{% /note %}}

![](/images/license.png)

---

### Провайдер выдачи лицензии

    {{% note %}}
Провайдер - это паттерн Стратегия, позволяющий инкапсулировать логику выдачи лицензий в отдельные модули/сервисы, так обращаясь с одними и теми-же методами мы уже умеем работать с выдачей лицензии через файл строгой структуры(лицензии JB) и через группы Active Directory
    {{% /note %}}

![](/images/provider.png)

---

### Фильтр

    {{% note %}}
Фильтр - набор условий позволяющий найти конкретный набор людей; Состоит из множества параметров, может быть безболезненно расширен новыми параметрами. Супервизорам системы дозволено сохранять фильтры с любыми названиями и по клику загружать их для просмотра сотрудников попадающих под фильтр 
    {{% /note %}}

![](/images/filters.png)

---

### Правило

    {{% note %}}
Правило - это сущность объединяет фильтр и лицензии, позволяя выдавать в автоматическом режиме сотрудникам, попадающим под фильтр, экземпляр лицензии 
    {{% /note %}}

![](/images/rules.png)

---

### Заявка на выдачу/отзыв

    {{% note %}}
Заявка - позволяет выдать или забрать лицензию у любого сотрудника 
    {{% /note %}}

![](/images/requests.png)


{{% /section %}}