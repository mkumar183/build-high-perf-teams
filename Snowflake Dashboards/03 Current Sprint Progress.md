![[Pasted image 20250328214420.png]]

![[Pasted image 20250328214451.png]]

Planned by EPIC: 
select 
epic_summary, team,
sum(case when csp.count_or_sp = 'count' then 1 else story_points end) count_or_sp
from JIRA_ISSUES_VIEW jv
inner join count_or_sp csp on csp.count_or_sp = :count_or_sp
where array_size(array_intersection(get_current_sprint(), sprints)) > 0
and team = :team
group by epic_summary,team

;
Status by Team: 
select status_ordered , fixversion_string, sum(case when csp.count_or_sp = 'count' then 1 else story_points end) count_or_sp
from JIRA_ISSUES_VIEW jv
inner join dim_sprints sp on sp.state = 'active' and array_contains(sp.sprint_name::variant, jv.sprints)
left outer join fact_jira_sprint_added_removed fjs on fjs.issue_key = jv.issue_key and fjs.sprint_name = sp.sprint_name
left outer join count_or_sp csp on csp.count_or_sp = :count_or_sp
where team = :team
and issue_type = :issuetype
group by status_ordered, fixversion_string
;

Completed vs Planned: 
select 
team, 
sum(count_or_sp) - sum(case when resolution_date is not null or status = 'Ready to Accept' then count_or_sp else 0 end) count_or_sp_pending,
sum(case when resolution_date is not null or status = 'Ready to Accept' then count_or_sp else 0 end) count_or_sp_completed
from (
    select 
    team, 
    (case when csp.count_or_sp = 'count' then 1 else story_points end) count_or_sp, 
    resolution_date, 
    status
    from JIRA_ISSUES_VIEW jv
    inner join count_or_sp csp on csp.count_or_sp = :count_or_sp
    where array_size(array_intersection(get_current_sprint(), sprints)) > 0
    and team = :team
)
group by team
;

By Epic by Team: 
select 
epic_summary, team,
sum(case when csp.count_or_sp = 'count' then 1 else story_points end) count_or_sp
from JIRA_ISSUES_VIEW jv
inner join count_or_sp csp on csp.count_or_sp = :count_or_sp
where array_size(array_intersection(get_current_sprint(), sprints)) > 0
and team = :team
group by epic_summary,team

;

Days Remaining: 
select datediff(day, current_date(), end_date ) from view_dim_sprints where state = 'active' and project_name = 'UIHN'

Tickets with High Cycle Time: 
select 
cs.age_days
, coalesce(TOTALDEVCYCLETIMEFORMATED, '0 days')  dev_cycle_time
, flagged_time_formated
, cs.status  
, cs.issue_url
, cs.department
, cs.sales_force_links
, (select sum(spent_hours) from view_dim_worklogs vdw where vdw.issue_key = cs.issue_key) as hours_logged
, cs.team
, cs.assignee
, array_size(cs.sprints) no_of_sprints
, cs.flagged
, cs.fixversions
, cs.summary 
, cs.epic_summary
, (select count(username) from view_dim_worklogs vdw where vdw.issue_key = cs.issue_key) as contributors
, cs.issue_type
, cs.sprints
, DEVINPROGRESSFORMATED dev_in_progress_time 
, READYFORCODEREVIEWFORMATED ready_for_cr_time 
, CODEREVIEWINPROGRESSFORMATED code_review_in_progress_time 
, READYFORTESTFORMATED ready_for_review_time 
, TESTINGINPROGRESSFORMATED testing_in_progress_time
,totaldevcycletime
 from jira_issues_view cs 
 where array_size(array_intersection(get_current_sprint(), sprints)) > 0
  and status = :status
  and team = :team
  -- and team in ('Interstaller','Spartans','Data-Ninjas','Pixel','Pinnacle','Othello','KBNext')
  and totaldevcycletime > 432000
order by totaldevcycletime desc, cs.status_order, cs.team ;

Summary By EPIC: 
with epics_in_sprint as (
select distinct epic_link
from JIRA_ISSUES_VIEW_v2 jiv 
inner join view_dim_sprints vds on vds.raw_sprint_name = get_current_sprint()[0] 
where array_size(array_intersection(get_current_sprint(), sprints)) > 0
and team = :team
), 
epic_sum as (
select epic_link, sum(story_points) sp, count(issue_key) cnt
from jira_issues_view_v2 jiv 
group by jiv.epic_link
)
select es.epic_link, jv.epic_summary
,sum(jv.story_points) sp_curr_sprint
,max(es1.sp) total_sp_epic
,jve.status epic_status 
from JIRA_ISSUES_VIEW_v2 jv 
inner join epics_in_sprint es on  jv.epic_link = es.epic_link
inner join epic_sum es1 on es1.epic_link = es.epic_link 
inner join jira_issues_view jve on jv.epic_link = jve.issue_key
inner join view_dim_sprints vds on vds.raw_sprint_name = get_current_sprint()[0] 
--left outer join count_or_sp csp on csp.count_or_sp = :count_or_sp
where array_size(array_intersection(get_current_sprint(), jv.sprints)) > 0
and jv.team = :team
group by jv.epic_summary , es.epic_link, jve.status
order by 3 desc
;


All ticket list: 
-- spilled over tickets 
select 
timestampdiff(day, sp.start_date, fjs.date_added_utc) added_days_after ,
issue_type, 
flagged,
summary,
status, 
assignee,
team,
sp.sprint_name,
epic_summary, 
story_points,
age_days,
TOTALDEVCYCLETIMEFORMATED dev_cycle_time,
flagged_time_formated flagged_time,
issue_url, 
sprints,
component_extn,
fixversions,
MERGE_REQUEST_URL,
COMMENT_WITH_GIT_COMMIT,
reporter,
department,
cast(status_order as number) status_order,
jv.resolution_date,
(select sum(spent_hours) from view_dim_worklogs vdw where vdw.issue_key = jv.issue_key) as hours_logged,
(select count(username) from view_dim_worklogs vdw where vdw.issue_key = jv.issue_key) as contributors
from JIRA_ISSUES_VIEW jv
inner join dim_sprints sp on sp.state = 'active' and array_contains(sp.sprint_name::variant, jv.sprints)
left outer join fact_jira_sprint_added_removed fjs on fjs.issue_key = jv.issue_key and fjs.sprint_name = sp.sprint_name
where team = :team
and issue_type = :issuetype
order by status_order, assignee
;

Pending: 
-- spilled over tickets 
select epic_summary, 
    issue_type as type,
    count(*) as counts,
    sum(story_points) as story_points,
    count(case when resolution_date is not null or status = 'Ready to Accept' then issue_key else null end) completed_count,
    sum(case when resolution_date is not null or status = 'Ready to Accept' then story_points else 0 end) completed
from JIRA_ISSUES_VIEW jiv 
inner join view_dim_sprints vds on vds.raw_sprint_name = get_current_sprint()[0] 
where array_size(array_intersection(get_current_sprint(), sprints)) > 0
and team = :team
group by epic_summary, issue_type
order by count(*) desc
;

jira comments:
select * from fact_jira_comments fjc 
inner join employee_data_view edv on edv.email_address = fjc.author_email
inner join view_dim_sprints vds on vds.raw_sprint_name = GET_CURRENT_SPRINT()[0]
where CONVERT_TIMEZONE('US/Pacific', 'UTC', created) between vds.start_date and vds.end_date
and edv.scrum_team = :team
; 

team counts:
select scrum_team, department, count(*) 
from employee_data_view edv 
where edv.scrum_team = :team
group by scrum_team, department


MRs updated:
select 
 case when contains(url, 'Clients') then 'custom' else vmr.project_name end project_name,  
 title,  
 description,
 author,
 assignees,
 reviewers,
 url, 
 merged_at,
 created_at,
 num_files,
 comment_count,
 num_lines_added,
 num_lines_removed,
 source_branch,
 target_branch,
 issue_impacted
from VIEW_MERGE_REQUESTS vmr 
inner join view_dim_sprints vds on vds.state = 'active' and vds.project_name = 'UIHN'
inner join employee_data_view edv on (vmr.versifit_email = edv.versifit_email or vmr.author = edv.github_user_name or vmr.author = edv.full_name)
where edv.scrum_team = :team
and vmr.updated_at between vds.start_date and vds.end_date
;


GIT Commits: 
select 
    fc.project_name, 
    title, 
    emp.scrum_team, 
    created_at, 
    web_url,  
    emp.scrum_team, 
    emp.full_name
from fact_commits fc
inner join view_dim_sprints vds on vds.state = 'active' and vds.project_name = 'UIHN'
left outer join employee_data_view emp on (fc.author_email = emp.versifit_email or fc.author_email = emp.github_user_name)
where created_at between vds.start_date and vds.end_date
and emp.scrum_team = :team
;

confluence contributions:
select space, title, version ver_updated, conf.contributors, createdby, page_createdby, page_createddate,  link, ded.scrum_team, conf.lastupdatedby
from VIEW_CONFLUENCE_CONTRIBUTORS conf 
left outer join employee_data_view ded on ded.email_address = conf.createdby
inner join view_dim_sprints vds on vds.raw_sprint_name = GET_CURRENT_SPRINT()[0]
where CREATEDDATE between vds.start_date and vds.end_date
and ded.scrum_team = :team
;

future sprint backlog: 
select issue_key, 
issue_type, 
status, 
epic_summary, 
story_points, 
summary, sprints, component_extn,  customer, reporter,  department, priority, sales_force_links, age_days, totaldevcycletime, total_flagged_time
from jira_issues_view jv
inner join view_dim_sprints sp on array_contains(sp.raw_sprint_name::variant, jv.sprints) and sp.future_sprint_order = 1
where jv.team = :team
and resolution_date is null
order by rank;

backlog beyond 3 sprints:
select issue_key, rank, summary, issue_url, issue_type, epic_summary, epic_url, priority, component_extn, fixversions, status, jv.sprints, assignee, age_days, totaldevcycletimeformated dev_cycle_time, total_flagged_time, updated_by_users_count, sales_force_links, district_or_customer, comment_count, participants, reporter, department, customer, labels, updated, capitalization
from jira_issues_view jv
-- inner join closed_and_active_sprints on  array_size(array_intersection(jv.sprints,closed_and_active_sprints.sprints)) = 0
where jv.team = :team
and issue_type in ('Bug', 'User Story')
and resolution_date is null and status not in  ('Ready to Accept')
--and priority not in ('3-Low') and priority is not null 
and (array_size(jv.sprints) = 0 or array_size(array_intersection(GET_FUTURE_AND_ACTIVE_SPRINTS(), sprints)) = 0)
order by rank asc;

