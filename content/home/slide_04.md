## Расчет выдачи лицензиий сотруднику

{{% section %}}

### Собираем всех активных физических лиц

{{% note %}}
	Просто берем  всех активных физиков из hr схемы
{{% /note %}}

```sql{}
	SELECT DISTINCT e.person_id
	FROM human_resource.persons p
			 JOIN human_resource.employees e ON e.person_id = p.id
		AND e.is_active
		AND e.for_staff
		AND (e.dismiss_date IS NULL OR e.dismiss_date::date >= now()::date)
		AND p.login != '';`
```

---

### По списку физиков мы джойним активных сотрудников

{{% note %}}
	Джойним сотрудников(инфу о должностях, отделах и тд)
{{% /note %}}

```sql{}
SELECT count(*) over ()       as total_count,
       p.id                   as profile_id,
       p.surname,
       p.name,
       p.patronymic,
       p.login,
       ...
       ARRAY(
               select t.name
               from (select unnest(o.vertical_ids) id) ids
                        join human_resource.org_structures t on t.id = ids.id
           )                  as org_vertical

FROM human_resource.persons p
         JOIN human_resource.employees e 
			ON e.person_id = p.id AND p.login != '' 
			AND CASE WHEN $2 THEN TRUE ELSE e.is_active END
         JOIN human_resource.org_structures o ON o.id = e.org_structure_guid
         LEFT JOIN human_resource.position_families pf on pf.id = e.position_family_id
         LEFT JOIN human_resource.position_levels pl on pl.id = e.position_level_id
         LEFT JOIN human_resource.company c on c.prefix = e.company_prefix
WHERE e.person_id = any ($1);`
```
    
---

### По списку сотрудников мы получаем кто подходит под какое правило

{{% note %}}
	Теперь мы снова id физика на основе правил, по факту мы джойним правила по их элементам к физикам с подджойненой инфой о сотрудниках, с ограничением на список айдишников которые мы получили раньше. Добавление нового условия в правила происходит как раз в этом месте. Добавить можно фильтрацию абсолютно все что есть в инфе о физике/сотруднике в hr схеме. 

	Правила работают по принципе IN-OR - OUTER-AND
{{% /note %}}

```sql{}
SELECT e.person_id,
		   coalesce(array_agg(distinct r.id), array[]::int[]) as rule_ids
	FROM human_resource.employees e
			 JOIN human_resource.org_structures o ON o.id = e.org_structure_guid and not o.is_deleted
			 JOIN rules r ON
			case
				when cardinality(r.position_names) > 0 then e.position = any (r.position_names)
				ELSE TRUE
				END
			AND case
					when cardinality(r.position_levels) > 0 then e.position_level_id = any (r.position_levels)
					ELSE TRUE
				END
			AND case
					when cardinality(r.position_families) > 0 then e.position_family_id = any (r.position_families)
					ELSE TRUE
				END
			AND case
					when cardinality(r.legal_entities) > 0 then e.company_prefix = any (r.legal_entities)
					ELSE TRUE
				END
			AND case
					when r.apply_to_children then o.vertical_ids && r.org_structure_ids
					else
						case
							when cardinality(r.org_structure_ids) > 0 then e.org_structure_guid = any (r.org_structure_ids)
							ELSE TRUE
							END
				END
			AND e.is_active
			AND r.status = $2
	
	WHERE e.person_id = any ($1)
	GROUP BY person_id;
```
    
---

### Получение списка персональных лицензий сотрудников
{{% note %}}
	Мы получаем список персональных лицензий которые выданы активным сотрудникам и которые находятся в статусах "Активна", "Ожидает активации", "Ожидает деактивации"
{{% /note %}}

```sql{}
SELECT 
		pl.person_id,
		pl.licence_id,
		pl.given_at,
		pl.status,
		pl.is_manual
	FROM personal_licences pl
	WHERE
		pl.person_id = any($1) AND pl.status = any($2);
```

---

### Проверка - есть ли активные запросы на получение лицензии
{{% note %}}
	Все по тому-же списку активных сотрудникв, собираем список заявок на получение лицензии.
{{% /note %}}

```sql{}
SELECT 
		id,
		licence_id,
		created_by,
		created_at,
		updated_at,
		subject_id,
		comment,
		status,
		coalesce(head_approver_id, 0) as head_approver_id,
		coalesce(supervisor_approver_id, 0) as supervisor_approver_id,
		request_type,
		reason
	FROM personal_requests
	WHERE 
		subject_id = any($1); /* в заявке subject_id это то КОМУ нужно выдать лицензию  */
```

---

### Формирование ProfileLicenceRelation для принятия решений
{{% note %}}
	ProfileLicenceRelation структура содержащая в себе информацию о физике, его сотрудниках и списки под какие правила он попадает, какие лицензии ему уже выданы и какие сейчас у него существуют запросы.
{{% /note %}}

```go{}
//Пихаем все наши полученные списки выше в такую модель
//Для каждого физика мы получаем такую
//индивидуальную структуру с данными
type ProfileLicenceRelation struct {
	Profile

	PersonalLicences []PersonalLicence
	PersonalRequests []PersonalRequest
	RelatedRules     []Rule
}

//Везде используется один ключ ID (staff-id физика)

plrs := make([]models.ProfileLicenceRelation, 0, len(profileIDs))

for _, p := range profiles {
	plrs = append(plrs,
		models.NewProfileLicenceRelation(
			p,
			personalLicences[p.ID],
			activeRequests[p.ID],
			rules[p.ID]))
}
```

---

### Принятие решения о выдаче (калькуляция)
{{% note %}}
	перебираем получившийся массив ProfileLicenceRelation и для каждой структуры вызываем CalculatePersonalLicences для того чтобы понять что выдать/отнять у сотрудника


	в CalculatePersonalLicences мы принимаем решение - выдавать лицензию или нет, а так-же какие лицензии отозвать если сотрудник перестал попадать под правила
{{% /note %}}

```go{}
	for _, pr := range profileRelations {
		personalLicences = append(personalLicences, pr.CalculatePersonalLicences()...)
	}
```

---

{{< figure src="/images/data/diagrams/process/CalculatePersonalLicences_1.svg"  height="500px">}}

---

{{< figure src="/images/data/diagrams/process/CalculatePersonalLicences_2.svg"  height="500px">}}

---

### Выдача персональной лицензии и ее активация

{{% note %}}
    записываем в базку в табличку персональных лицензий выданные экземпляры

	кроной мы выбираем все ПЛ которые в статусах ожидает активации/деактивации
{{% /note %}}

![](/images/queue.png)

{{% /section %}}