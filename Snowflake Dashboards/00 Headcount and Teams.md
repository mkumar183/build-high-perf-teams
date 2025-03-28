![[Pasted image 20250328162439.png]]


select company_code, department, count(*) from employee_data_view
where active = true
and rd_product in ('Unified Insights','Kickboard')
and scrum_team != 'Gravity'
group by company_code, department
;

select company_code || '-' || DEPARTMENT , 
job_title_career_ladder,
job_title,
department, count(*) from employee_data_view
where active = true
--and rd_product in ('Unified Insights','Kickboard')
and scrum_team != 'Gravity'
group by company_code, job_title, department, job_title_career_ladder
;

select job_title_career_ladder,  count(*) from employee_data_view
where active = true
--and rd_product = 'Unified Insights'
and scrum_team != 'Gravity'
and active = true 
and company_code like 'IN%'
group by job_title_career_ladder
;

select scrum_team, department, count(*) from employee_data_view
where active = true
--and rd_product = 'Unified Insights'
and scrum_team != 'Gravity'
group by scrum_team, department
;

select  employee_number, full_name, department, company_code, rd_product, scrum_team, job_title, email_address, versifit_email, supervisor_name
from employee_data_view
where active = true
and company_code = 'INPWR'
and rd_product in ('Unified Insights', 'Kickboard')
;

select rd_product, department || '-' || company_code, count(*) from employee_data_view
where active = true
--and rd_product = 'Unified Insights'
and scrum_team != 'Gravity'
group by rd_product, department, company_code
;

select supervisor_name, count(*) from employee_data_view
where active = true
--and rd_product = 'Unified Insights'
and scrum_team != 'Gravity'
group by supervisor_name
;
