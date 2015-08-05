# -
追特定用户近一年订单历史
*3.1.1创建浏览申请页面的用户的名单;
use cf;
create table cf.apply_browse_150717 as 
select * from
dm. d_fact
where 
(dt between "2015-06-04" and "2015-07-17") and current_page_url like "http://HUHSJSKSKAAK.html";
*3.1.2将给定的id\dm. _h的end_dt='4712-12-31'分区表\dm. d_sum近一年的分区表内连接,
得到id、用户号、真实姓名\城市\注册时间、订单等等的信息;
create table cf.org_order_150717
as select a.id,b.user_id,b.city,b.reg_time,
c.parent_sale_ord_id,c.item_first_cate_name, c.item_third_cate_name, 
c.sale_qtty, c.user_actual_pay_amount,c.cnee_mobile_no, c.rev_addr,c.place_ord_ip,c.dt
from
cf.apply_browse_150717 a
join dm. _h b
on a.id=b.id
join dm. _d_sum c
on a.id=c.user_log_acct
where b.user_level in (64,65,90)
and b.end_dt='4712-12-31'
and c.complete_flag=1 and c.virtual_ord_flag=0
and (c.dt between '2014-07-16'and'2015-07-17');

*3.1.3  20150716这段先计算每个id的注册时间、帐龄、城市、订单数、订单金额、客单价、商品数量、购买天数等信息;
create table cf.org_buyinfo_150717 as 
select user_id,id,
to_date(reg_time) as regtime,
datediff('2015-07-17',to_date(reg_time))/30 as id_age_month,
city,
count(distinct parent_sale_ord_id) as order_num,
sum(user_actual_pay_amount) as order_amount,
sum(user_actual_pay_amount)/count(distinct parent_sale_ord_id) as avg_order_amount,
sum(sale_qtty) as item_num,
count(distinct dt) as buy_days,
count(distinct cnee_mobile_no) as mobile_no,
count(distinct rev_addr) as rev_addr_no,
count(distinct place_ord_ip) as ip_no
from cf.org_order_150717
group by user_id,id,to_date(reg_time),datediff('2015-07-17',to_date(reg_time))/30,city;
 
*3.1.4  五表相连，最终创建加了按购买金额、商品数量排名的一级、三级品类的表;
create table cf.browse_final_150717 as 
select a.*,b.money_first_cate_name,c.num_first_cate_name,d.money_third_cate_name,e.num_third_cate_name
from
cf.org_buyinfo_150717 a
join (select id,item_first_cate_name as money_first_cate_name,
sum(user_actual_pay_amount) as order_amount,
row_number()over(partition by id
                 order by sum(user_actual_pay_amount) desc
                 ) as rank
from cf.org_order_150717
group by id,item_first_cate_name) b
on a.id=b.id
join (select id,item_first_cate_name as num_first_cate_name,
sum(sale_qtty) as item_num,
row_number()over(partition by id
                 order by sum(sale_qtty) desc
                 ) as rank
from cf.org_order_150717
group by id,item_first_cate_name) c
on a.id=c.id
join (select id,item_third_cate_name as money_third_cate_name,
sum(user_actual_pay_amount) as order_amount,
row_number()over(partition by id
                 order by sum(user_actual_pay_amount) desc
                 ) as rank
from cf.org_order_150717
group by id,item_third_cate_name) d
on a.id=d.id
join (select id,item_third_cate_name as num_third_cate_name,
sum(sale_qtty) as item_num,
row_number()over(partition by id
                 order by sum(sale_qtty) desc
                 ) as rank
from cf.org_order_150717
group by id,item_third_cate_name) e
on a.id=e.id
where b.rank='1' and c.rank='1'and d.rank='1' and e.rank='1';

