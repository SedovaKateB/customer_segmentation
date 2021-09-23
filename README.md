Задачи:
нужно провести исследование, отличаются ли продажи и retention пользователей из России от всех остальных пользователей. А также выделить core-игроков для дальнейшего исследования их поведения.

Конкретные шаги:
Определить динамику продаж по сегментам «Россия»/«Не Россия».
Разобраться, отличается ли месячный retention пользователей России от retention пользователей из других стран. 
Выделить core-игроков, а также посмотреть, насколько отличается ежемесячный доход с них от дохода с обычных игроков.

1 таблица: purchases_per_countries (Динамика продаж Россия/не Россия)
with users_with_geo as
(select
u.id,
case
	when country ='Russia' then 'yes'
	when country !='Russia' then 'no'
	when LENGTH(u.phone::text)=11 and left (u.phone::text,1)='7' then 'yes'
	when LENGTH(u.phone::text)!=11 and left (u.phone::text,1)!='7' and
 	u.phone::text is not NULL then 'no'
	when country is not NULL or phone is not NULL then 'no'
	else 'unknown'  end is_russia
from gd2.users1 u left JOIN gd2.addresses ad ON u.id=ad.user_id
)
select
amount,
created_at,
p.id,
state,
p.user_id,
is_russia
from gd2.purchases p JOIN users_with_geo uwg ON p.user_id=uwg.id
where state='successful' and is_russia<>'unknown'

2 и 3 таблицы: Когорта Россия/не Россия (cohorts_russia/cohorts_not_russia)
with users_with_geo as
    (select
		u.id,
		case
		when country ='Russia' then 'yes'
		when country !='Russia' then 'no'
		when LENGTH(u.phone::text)=11 and left (u.phone::text,1)='7' then 'yes'
		when LENGTH(u.phone::text)!=11 and left (u.phone::text,1)!='7' and
		u.phone::text is not NULL then 'no'
		when country is not NULL or phone is not NULL then 'no'
		else 'unknown'  end is_russia
	from 
	gd2.users1 u left JOIN gd2.addresses ad ON u.id=ad.user_id),
first_purchases as
    (select
         distinct p.user_id,
         min(date_trunc('month', p.created_at)) min_date
    from
        gd2.purchases p
    where
        p.state='successful'
    group by
        1
    order by
        2),
reg_purchases as
    (select
        uwg.id,
        date_trunc('month', p.created_at) month_purchase, 
        count(distinct case when uwg.is_russia='yes' then p.id end) russia_purchase, 
        count(distinct case when uwg.is_russia='no' then p.id end ) other_purchase
    from
        users_with_geo uwg 
    join
        gd2.purchases p
            on
                uwg.id=p.user_id and p.state='successful'
    group by
        1,2)
select 
        fp.min_date::date,
        rp.month_purchase::date,
		uwg.is_russia,
		count(distinct fp.user_id)
    from
        first_purchases fp 
    left join users_with_geo uwg on fp.user_id=uwg.id
    join reg_purchases rp on rp.id=fp.user_id
 group by 1,2,3

4 таблица: revenues_by_segments (пользователи и доход по сегментам)
with avg_bill_june as
    (select distinct 
        p.user_id,
         count(p.amount) purchase_counter,
        avg(p.amount) avg_amount,
         sum(p.amount) revenue
    from
         gd2.purchases p
    where
        p.state='successful'
        and
         p.created_at
             between
                 '2020-06-01'
                 and
                 '2020-06-30'
    group by
         1)
select
    abj.user_id,
    abj.revenue,
    (case
         when
             abj.avg_amount > 750
         then
             'core'
         else
             'not_core'
         end) as segment
from
    avg_bill_june abj
order by
    segment

https://app.powerbi.com/reportEmbed?reportId=b8aa736c-157e-4cac-85b2-d9682eb41a68&autoAuth=true&ctid=6a4dee01-c3f5-4d4b-bdd2-9e1f1482ac5d&config=eyJjbHVzdGVyVXJsIjoiaHR0cHM6Ly93YWJpLXdlc3QtZXVyb3BlLWQtcHJpbWFyeS1yZWRpcmVjdC5hbmFseXNpcy53aW5kb3dzLm5ldC8ifQ%3D%3D
