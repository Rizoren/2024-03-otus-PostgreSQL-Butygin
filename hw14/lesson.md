> Занятие 14 
Триггеры, поддержка заполнения витрин.

Выполнен скрипт: hw_triggers.sql и содан тригер на таблицу продаж.

```sql
CREATE OR REPLACE FUNCTION pract_functions.sale_logger()
	RETURNS trigger
	LANGUAGE plpgsql
AS $function$
DECLARE 
	l_n_good pract_functions.goods;
	l_o_good pract_functions.goods;
	l_sum pract_functions.good_sum_mart.sum_sale%type;
begin		
	case (TG_OP)
		when 'INSERT' then
		    select * into l_n_good 
		      from pract_functions.goods 
		     where goods_id = new.good_id;
		    
		    update pract_functions.good_sum_mart set sum_sale = sum_sale + (l_n_good.good_price * new.sales_qty)
		    where good_name = l_n_good.good_name;
		    if found then return null; end if;
		    insert into pract_functions.good_sum_mart (good_name, sum_sale) values(l_n_good.good_name, l_n_good.good_price * new.sales_qty);
		when 'UPDATE' then
			if new <> old then
			    select * into l_n_good 
			      from pract_functions.goods 
			     where goods_id = new.good_id;
			    if new.good_id <> old.good_id then
				    select * into l_o_good 
				      from pract_functions.goods 
				     where goods_id = old.good_id;
				    
				    update pract_functions.good_sum_mart set sum_sale = sum_sale - (l_o_good.good_price * old.sales_qty)
				where good_name = l_o_good.good_name
			    returning sum_sale into l_sum;
			    if l_sum <= 0 then
				delete from pract_functions.good_sum_mart where good_name = l_o_good.good_name;
			    end if;
			   
				update pract_functions.good_sum_mart t set sum_sale = sum_sale + (l_n_good.good_price * new.sales_qty)
				where good_name = l_n_good.good_name;
				if found then return null; end if;
				insert into pract_functions.good_sum_mart (good_name, sum_sale) values(l_n_good.good_name, l_n_good.good_price * new.sales_qty);				     	
				else
				    update pract_functions.good_sum_mart set sum_sale = sum_sale - (l_n_good.good_price * old.sales_qty) + (l_n_good.good_price * new.sales_qty)
				where good_name = l_n_good.good_name;
				end if;
			end if;
		when 'DELETE' then	
		    select * into l_o_good 
		      from pract_functions.goods 
		     where goods_id = old.good_id;
		    
		    update pract_functions.good_sum_mart set sum_sale = sum_sale - (l_o_good.good_price * old.sales_qty)
		     where good_name = l_o_good.good_name
		    returning sum_sale into l_sum;
		   
		    if l_sum <= 0 then
			delete from pract_functions.good_sum_mart where good_name = l_o_good.good_name;
		    end if;
	end case;

	return null; 
END;
$function$
;

DROP TRIGGER IF EXISTS sale_aiud_trg ON pract_functions.sales;
CREATE TRIGGER sale_aiud_trg
AFTER INSERT OR UPDATE OR DELETE 
ON pract_functions.sales
FOR EACH ROW
EXECUTE PROCEDURE pract_functions.sale_logger();
```

В таком примитивном варианте приемуществ не то что бы много будет у триггера. Но если в системе появится историзм цен, то запрос для получения отчета усложнится... (но скорее всего не достаточно сильно), а вот если еще появится система скидок и т.п. то посчитать в моменте 1 запись гораздо проще (тем более что место расчета всегда может быть изменено с большей степенью свободы, это же функция для триггера, в отличии от статичного запроса у отчета) 
