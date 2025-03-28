![[Pasted image 20250328163514.png]]

incoming by dept logged:
select department, component_extn, sum(created), sum(resolved) 
from (
    select coalesce(department, 'None') department, component_extn,  count(*) created, 0 resolved 
    from jira_issues_view issues
    where issue_type = :issuetype 
    and department = :department
    and CONVERT_TIMEZONE('US/Pacific', 'UTC', created) >= timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP())
    group by department, component_extn
    union all
    select coalesce(department, 'None') department, component_extn, 0 created , count(*) resolved 
    from jira_issues_view issues
    where issue_type = :issuetype 
    and department = :department
    and CONVERT_TIMEZONE('US/Pacific', 'UTC', resolution_date) >= timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP())
    group by department, component_extn
) 
group by department, component_extn

by component:
select component_extn, sum(created) created, sum(resolved) resolved 
from (
select coalesce(component_extn, 'None') component_extn, count(*) created, 0 resolved 
from jira_issues_view issues
where issue_type = :issuetype
and department = :department
and CONVERT_TIMEZONE('US/Pacific', 'UTC', created) >= timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP())
group by component_extn
union all 
select coalesce(component_extn, 'None') component_extn, 0 created, count(*) resolved
from jira_issues_view issues
where issue_type = :issuetype
and department = :department
and CONVERT_TIMEZONE('US/Pacific', 'UTC', resolution_date) >= timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP())
group by component_extn
) 
group by component_extn
;

closure by team: 
select team, sum(created) created, sum(resolved) resolved 
from (
select coalesce(team, 'None') team, count(*) created, 0 resolved 
from jira_issues_view issues
where issue_type = :issuetype
and department = :department
and CONVERT_TIMEZONE('US/Pacific', 'UTC', created) >= timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP())
group by team
union all 
select coalesce(team, 'None') team, 0 created, count(*) resolved
from jira_issues_view issues
where issue_type = :issuetype
and department = :department
and CONVERT_TIMEZONE('US/Pacific', 'UTC', resolution_date) >= timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP())
group by team
) 
group by team
;

by customer:
select issues.sf_customer_account_name , count(*) created, 0 resolved 
from jira_issues_view_v2 issues
where issue_type = :issuetype
and department = :department
and CONVERT_TIMEZONE('US/Pacific', 'UTC', created) >= timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP())
and sales_force_links is not null
group by sf_customer_account_name;

incoming x hours:
select issue_key
, department
, labels
, coalesce(sf_customer_account_name, customer_code) customer 
, summary
, assignee
, components
, component_extn
, fixversions
, sales_force_links
, get_jira_url(issues.issue_key)
, status
, issue_type
, reporter
, epic_summary
, team
, description
, sprints
, comment_count
, created
from jira_issues_view issues
where issue_type = :issuetype
and issue_type = :issuetype
and CONVERT_TIMEZONE('US/Pacific', 'UTC', created) >= timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP())
and department = :department
order by created desc
;

-- Resolved x hours 
select 
 summary, resolution
 , fixversion_string
 , case when content_commit_url is null and merge_request_url is null then 'N' else 'Y' end has_mr_or_commit
, recommendation_for_avoidance
, assignee
, team
, get_jira_url(issues.issue_key) issue_url
, age
, totaldevcycletimeformated dev_cyc_time
, comment_count
, array_size(participants) part_count
, issues.component_extn
, worklog_spent_hours
, brief_fix_description
, brief_problem_description
, was_issue_avoidable
, merge_request_url
, content_commit_url
, issue_type
, department
, sales_force_links
, status
, epic_summary
, department
, reporter
, description
, sprints
, comment_count
, devinprogressformated
, totaldevcycletimeformated
, total_flagged_time
from jira_issues_view issues
where issue_type in ('Bug' , 'User Story')
and team = :team
and department = :department
and issue_type = :issuetype
and epic_link not in ('UIHN-2093')
and CONVERT_TIMEZONE('US/Pacific', 'UTC', resolution_date) >= timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP())
order by coalesce(dev_cyc_time, '0') desc;


MRs 
select project_name, title, issue_impacted, author, scrum_team, url,
case when created_at < to_date(updated_at) then 'Updated' else 'New' end upd_or_new, 
num_files, num_lines_added, num_lines_removed, contains_normalizejs, comment_count, source_branch, target_branch,  versifit_email
url, assignees, reviewers
from view_merge_requests 
where CONVERT_TIMEZONE('US/Pacific', 'UTC', updated_at) >= timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP()); 

Git Commits:
select project_name, title, fc.issue_impacted , fc.author_scrum_team ,  created_at, web_url,  fc.author_full_name
from view_fact_commit fc
where created_at > timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP());
;

Confluence Contributors:
select space, 
title, 
version ver_updated, 
conf.contributors, 
createdby, 
page_createdby, 
page_createddate,  
link, 
ded.scrum_team, 
conf.lastupdatedby
from VIEW_CONFLUENCE_CONTRIBUTORS conf 
left outer join employee_data_view ded on ded.email_address = conf.createdby
where CREATEDDATE > timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP())
;

Comments:
select get_jira_url(fc.issue_key), edv.scrum_team, fc.author_name, fc.comment_body, fc.url
from fact_jira_comments fc 
left outer join EMPLOYEE_DATA_VIEW EDV on fc.author_email = edv.email_address
where CONVERT_TIMEZONE('US/Pacific', 'UTC', created) >= timeadd(hour, (select hours from x_hours where hours = :hours), CURRENT_TIMESTAMP())

; 