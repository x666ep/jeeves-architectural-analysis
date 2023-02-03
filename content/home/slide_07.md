## Обработка данных по сотрудникам

{{% section %}}

- Обработка данных происходит раз в 3 минуты в кроне
- Процесс происходит последовательно

---

### ProcessHireChanges

---

{{% note %}}
processHireChanges
1. Идем в БД, берем все employee_guid у которых есть необработанные приказы (SELECT DISTINCT)
2. В цикле перебираем этих сотрудников и получаем по нему его приказы(все в тч и не обработанные)
3. Вычисляем, применяем новое сотояние если дата события приказа наступила (возможные состояния это Работает ИЛИ Уволен) кладем это все в employee_hire_status_actual 

кратко:
- Смотрим приказы среди потока проведений/отмен и выбираем актуальное состояние, которое помещаем в таблицу актуальных приказов. (db_staff_api.human_resource.employee_hire_status_actual)
{{% /note %}}

![](/images/data/diagrams/staff-core/ProcessHireChangesFlow.svg)

---

### ProcessEmployeeUpdates

---

{{% note %}}
1. Оформление

- Берем из таблицы employee_hire_status_actual все приказы с типом "оформление" у которых дата оформления is null и джойним к ним последнии изменения из EmployeeLastChangesRaw и у которых Отсутствует флаг обработки событий из EmployeeLastChangesRaw

- По каждому сотруднику смотрим есть ли у него ФЛ(по СИДУ, если не нашли то по guid ФЛ), делаем UPSERT ФЛ и сотрудника (сотрудник будет неактивен - is_active = false) 
   
- Формируем событие оформления (EmployeeHireFormalized) 

- помечаем что приказ на найм - оформлен (дата оформления ставится текущей датой) 

{{% /note %}}

![](/images/data/diagrams/staff-core/ProcessEmployeeUpdatesPg1.svg)

---

{{% note %}}
1. Обновление

- Берем из таблицы employee_hire_status_actual все приказы с типом "оформление" у которых дата оформления is NOT null и джойним к ним последнии изменения из EmployeeLastChangesRaw и у которых Отсутствует флаг обработки событий из EmployeeLastChangesRaw

- По каждому сотруднику смотрим есть ли у него ФЛ(по СИДУ, если не нашли то по guid ФЛ) - если его нет, формируем ошибку, если ФЛ найдено то обновляем данные по нему если что-то изменилось 

- Проверяем есть ли ФЛ руководителя этого сотрудника - если оно не найдено, мы его скипаем до след итерации (рукль может появиться позже чем сотрудник) 

- Обновляем сотрудника, в тч активируем если он был деактивирован

- формируем событие EmployeeHired 

- Помечаем данные о сотруднике обработанными 

{{% /note %}}

![](/images/data/diagrams/staff-core/ProcessEmployeeUpdatesPg2.svg)

---

### ProcessAccountChanges

---

{{% note %}}
Весь механизм процесса такой же как у ProcessEmployeeUpdates
{{% /note %}}

![](/images/data/diagrams/staff-core/ProcessAccountChanges.svg)

---

### ProcessGradeChanges

---

{{% note %}}
processGradeChanges
1. Из таблицы актуальных грейдов, берем самые актуальные на текущий момент
2. Смотрим, отличаются ли актуальные цифры от того что записано в табличку employees
3. Если нашли разницу, обновляем эти данные в таблице, шифруем грейд перед отправкой события в кафку
4. отправляем событие
{{% /note %}}

![](/images/data/diagrams/staff-core/ProcessGradeChanges.svg)

---

### ProcessCBParams

---

{{% note %}}
processCBParams

это то что мы забираем из ручек 1С 
(Бонусы, Семейства должностей, Группы KPI)

1. Идем в табличку changes
2. Ищем не примененные ченджи
3. Отправляем ивент в кафку и сохраняем в employees новые значения
{{% /note %}}

![](/images/data/diagrams/staff-core/ProcessCBParams.svg)


{{% /section %}}