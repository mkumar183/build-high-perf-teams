![[Pasted image 20250328215738.png]]

![[Pasted image 20250328215814.png]]

![[Pasted image 20250328215834.png]]

Velocity Breakdown by EPICs: 
select jv.team, epic_summary,  sum(case csp.count_or_sp when 'count' then 1 else story_points end) count_or_sp
from jira_issues_view jv
inner join view_dim_sprints sp on sp.sprint_name = :sprint 
inner join count_or_sp csp on csp.count_or_sp = :count_or_sp 
left outer join fact_jira_sprint_added_removed fjs on fjs.issue_key = jv.issue_key and fjs.sprint_name = sp.raw_sprint_name
where array_contains(sp.raw_sprint_name:: variant, sprints)
  and team = :team
  and jv.resolution_date between sp.start_date and sp.completion_date
  and issue_type = :issuetype
group by jv.team, epic_summary
;


completion vs planned:
select team, sum(completed), sum(planned)
from (
select jv.team, sum(case when csp.count_or_sp = 'count' then 1 else story_points end) planned, 0 completed 
    from jira_issues_view jv
    inner join view_dim_sprints sp on sp.sprint_name = :sprint 
    inner join fact_jira_sprint_added_removed js on js.issue_key = jv.issue_key
    inner join count_or_sp csp on csp.count_or_sp = :count_or_sp
    where team = :team
      and js.sprint_name = sp.raw_sprint_name
      and date_added_utc <= timeadd(hour,0,sp.start_date) -- a grace time of 24 hours is added. if tickets is added 12 hours post sprint start its considered planned  is null)
      and (date_removed_utc >= timeadd(hour,0,sp.start_date) -- either its removed after sprint started 
      -- and date_added_utc <= sp.start_date -- a grace time of 24 hours is added. if tickets is added 12 hours post sprint start its considered planned  is null)
      -- and (date_removed_utc >= sp.start_date -- either its removed after sprint started 
      or date_removed_utc is null -- or its never removed 
           or date_removed_utc <= date_added_utc) -- this is no more required since we are ensure removed date is >= added date while populating data
    group by jv.team
    union all
    select jv.team, 0 planned, sum(case when csp.count_or_sp = 'count' then 1 else story_points end) completed
    from jira_issues_view jv
    inner join view_dim_sprints sp on sp.sprint_name = :sprint 
    inner join count_or_sp csp on csp.count_or_sp = :count_or_sp
    where array_contains(sp.raw_sprint_name:: variant, sprints)
      and team = :team
      and jv.resolution_date between sp.start_date and sp.completion_date 
    group by jv.team 
) group by team     
    ;

By Issue Types: 
select jv.team,  issue_type, sum(case csp.count_or_sp when 'count' then 1 else story_points end )  count_or_sp
from jira_issues_view jv
inner join view_dim_sprints sp on sp.sprint_name = :sprint 
inner join count_or_sp csp on csp.count_or_sp = :count_or_sp
where array_contains(sp.raw_sprint_name:: variant, sprints)
  and team = :team
  and issue_type = :issuetype
  and jv.resolution_date between sp.start_date and sp.completion_date
group by jv.team, issue_type
;

By Team by FixVersion;
select jv.team, (case fixversion_string when '' then 'Empty' else fixversion_string end) fixversion,  sum(case csp.count_or_sp when 'count' then 1 else story_points end) count_or_sp
from jira_issues_view jv
inner join dim_sprints sp on sp.sprint_name = :sprint 
inner join count_or_sp csp on csp.count_or_sp = :count_or_sp
where array_contains(sp.sprint_name:: variant, sprints)
  and team = :team
  and jv.resolution_date between sp.start_date and sp.completion_date
  and issue_type = :issuetype
group by jv.team, fixversion
;

With MR with No MR: 
select jv.team, 
(case when merge_request_url is not null then 'has mr' when comment_with_git_commit is not null then 'content commit' else 'None' end) git 
,sum(case csp.count_or_sp when 'count' then 1 else story_points end) count_or_sp
from jira_issues_view jv
inner join dim_sprints sp on sp.sprint_name = :sprint 
inner join count_or_sp csp on csp.count_or_sp = :count_or_sp
left outer join fact_jira_sprint_added_removed fjs on fjs.issue_key = jv.issue_key and fjs.sprint_name = sp.sprint_name
where array_contains(sp.sprint_name:: variant, sprints)
  and team = :team
  and jv.resolution_date between sp.start_date and sp.completion_date
  and issue_type = :issuetype
group by jv.team, git
;

By Team by Assigned: 
select jv.team, jv.assignee,  sum(case csp.count_or_sp when 'count' then 1 else story_points end) count_or_sp
from jira_issues_view jv
inner join dim_sprints sp on sp.sprint_name = :sprint 
inner join count_or_sp csp on csp.count_or_sp = :count_or_sp
left outer join fact_jira_sprint_added_removed fjs on fjs.issue_key = jv.issue_key and fjs.sprint_name = sp.sprint_name
where array_contains(sp.sprint_name:: variant, sprints)
  and team = :team
  and jv.resolution_date between sp.start_date and sp.completion_date
  and issue_type = :issuetype
group by jv.team, jv.assignee

Completed Issues: Review Fix Version, MR Link, RCA, Assignee, Status, Cycle Times
select jv.issue_key 
, timestampdiff(day, sp.start_date, fjs.date_added_utc) added_days_after 
, coalesce(sf_customer_account_name, customer_code) customer
, summary
, team
, epic_summary
, story_points
, totaldevcycletimeformated dev_cycle_time
, age
, flagged_time_formated flagged_time
, sp.sprint_name
, issue_url
, component_extn
, fixversion_string
, jv.comment_with_git_commit
, jv.merge_request_url
, has_no_fix_version
, issue_type
, recommendation_for_avoidance
, brief_fix_description
, brief_problem_description
, department
, sales_force_links
, devinprogressformated
, formated_cycle_time
, status
, sprints
, jv.resolution_date
, sp.start_date
, sp.completion_date
, story_points
, assignee
, fjs.date_added_utc
, fjs.date_removed_utc
, totaldevcycletime
from jira_issues_view jv
inner join dim_sprints sp on sp.sprint_name = :sprint 
left outer join fact_jira_sprint_added_removed fjs on fjs.issue_key = jv.issue_key and fjs.sprint_name = sp.sprint_name
where array_contains(sp.sprint_name:: variant, sprints)
  and team = :team
  and jv.resolution_date between sp.start_date and sp.completion_date
  and issue_type = :issuetype
  and department = :department
  order by coalesce(jv.totaldevcycletime,0) desc;

Spilled Over, Review # of sprints, Added after days, Status
-- Issues Not Completed
select issue_key , timestampdiff(day, start_date, time_added) added_days_after ,summary, team, status, sprints, issue_url, resolution_date, start_date, completion_date, 
time_added
from (
    select issue_key , summary, team, status, sprints, issue_url, jv.resolution_date, sp.start_date, sp.completion_date
    ,(select max(CONVERT_TIMEZONE('America/Los_Angeles', 'UTC',created_datetime)) 
       from FACT_STATUS_FULL_HISTORY hist 
       inner join dim_sprints sp on sp.sprint_name = :sprint 
       where hist.issuekey = jv.issue_key 
        and field_name = 'Sprint' 
        and contains(to_string, sp.sprint_name) 
        and not contains(coalesce(from_string,''), sp.sprint_name) 
    ) as time_added
    from jira_issues_view jv
    inner join dim_sprints sp on sp.sprint_name = :sprint 
    where array_contains(sp.sprint_name:: variant, sprints)
      and team = :team
      and (jv.resolution_date is null or resolution_date > sp.completion_date )
)
order by issue_key;

Issues Removed from Sprint
select distinct jv.issue_key, jv.team, jv.fixversions, jv.age, jv.sprints, jv.status, jv.issue_url, jv.summary, jv.epic_summary
from jira_issues_view jv
  inner join FACT_STATUS_FULL_HISTORY hist1 on jv.issue_key = hist1.issuekey
  inner join FACT_STATUS_FULL_HISTORY hist2 on jv.issue_key = hist2.issuekey
  inner join dim_sprints sp on sp.sprint_name = :sprint 
       where not contains(jv.sprints::variant, sp.sprint_name)  
        and hist1.field_name = 'Sprint' 
        and jv.team = :team
        and contains(hist1.to_string, sp.sprint_name)
        and not contains(coalesce(hist1.from_string,''), sp.sprint_name)
        and hist1.created_datetime <= sp.completion_date
        and contains(hist2.from_string, sp.sprint_name)
        and not contains(coalesce(hist2.to_string,''), sp.sprint_name)
        and hist2.created_datetime between sp.start_date and sp.completion_date
order by issue_key    

Last 5 Sprint Velocity (Will not work if multiple sprints selected)
select team, sp.sprint_name, 'planned' p_or_c,  sum(story_points), count(jv.issue_key)
    , sum(case when issue_type = 'Bug' then 1 else 0 end) bug_count
    , sum(case when issue_type = 'User Story' then 1 else 0 end) User_Story_count
from jira_issues_view jv
inner join view_dim_sprints sp1 on sp1.sprint_name = :sprint 
inner join view_dim_sprints sp on sp.future_sprint_order between sp1.future_sprint_order -5 and sp1.future_sprint_order and sp1.project_name = sp.project_name
inner join fact_jira_sprint_added_removed js on js.issue_key = jv.issue_key
where team = :team
  and js.sprint_name = sp.raw_sprint_name
  and date_added_utc <= timeadd(hour,24,sp.start_date) -- a grace time of 24 hours is added. if tickets is added 12 hours post sprint start its considered planned  is null)
  and (date_removed_utc >= timeadd(hour,24,sp.start_date) -- either its removed after sprint started 
       or date_removed_utc is null -- or its never removed 
       or date_removed_utc <= date_added_utc) -- this is no more required since we are ensure removed date is >= added date while populating data
group by sp.sprint_name, team
union all
select sp.sprint_name, team, 'completed',  sum(story_points), count(issue_key)
    , sum(case when issue_type = 'Bug' then 1 else 0 end) bug_count
    , sum(case when issue_type = 'User Story' then 1 else 0 end) User_Story_count
from jira_issues_view jv
inner join view_dim_sprints sp1 on sp1.sprint_name = :sprint 
inner join view_dim_sprints sp on sp.future_sprint_order between sp1.future_sprint_order -5 and sp1.future_sprint_order and sp1.project_name = sp.project_name
where array_contains(sp.raw_sprint_name:: variant, sprints)
  and team = :team
  and jv.resolution_date between sp.start_date and sp.completion_date 
group by sp.sprint_name , team
order by  team, sprint_name desc ,p_or_c desc;

scorecard:
select edv.full_name --, flm.sprint
, sum(coding_days) coding_days, 
                     sum(prs_submitted) prs_submitted, 
                     sum(prs_reviewed) prs_reviewed, 
                     sum(hloc_noramalized) hloc, 
                     sum(tickets_completed) tickets_completed, 
                     sum(velocity) velocity, 
                     sum(impact)/count(distinct sprint) impact,                      
                     sum(efficiency)/count(distinct sprint) efficiency,
                     round(avg(coding_days) *.05 + sum(prs_submitted) *.10 + sum(prs_reviewed) *.15 + sum(hloc_noramalized)/100.0 *.15 
                     + sum(tickets_completed) *.10 + sum(velocity) *.10 + avg(impact) *.20 + avg(efficiency) *.15) pscore
                from view_flow_metrics flm
                inner join employee_data_view edv on flm.email = edv.email_address
                left outer join view_dim_sprints vds on flm.sprint = vds.raw_sprint_name
                where edv.scrum_team = :team
                and sprint = :sprint
                and (sprint like '%UIHN%' or sprint like '%KB%')   
                -- and vds.start_date > DATEADD(DAY, -90, CURRENT_DATE())
                group by edv.full_name --, flm.sprint, vds.start_date
                order by edv.full_name; --, vds.start_date;

select edv.full_name, count(issue_key), sum(story_points)
  from jira_issues_view jv 
  inner join employee_data_view edv on jv.assignee_email = edv.email_address
  left outer join view_dim_sprints vds on vds.raw_sprint_name = :sprint
  where array_contains(vds.raw_sprint_name::variant, sprints)
  and edv.scrum_team = :team
  and vds.sprint_name = :sprint
 and resolution_date between vds.start_date and vds.end_date
 group by edv.full_name;

 

All ticket list:
-- spilled over tickets 
select 
issue_type, 
flagged,
summary,
status, 
assignee,
team,
epic_summary, 
story_points,
devinprogressformated,
flagged_time_formated,
issue_url, 
sprints,
component_extn,
fixversions,
reporter,
department,
cast(status_order as number) status_order
from JIRA_ISSUES_VIEW issues
inner join dim_sprints sp on sp.sprint_name = :sprint 
where array_contains(sp.sprint_name:: variant, sprints)
and team = :team
order by status_order, assignee
;

merge requests:
select project_name, 
       title, 
       issue_impacted, 
       author, 
       vmr.scrum_team, 
       url,
       case when created_at < to_date(updated_at) then 'Updated' else 'New' end upd_or_new, 
       num_files, 
       num_lines_added, 
       num_lines_removed, 
       contains_normalizejs, 
       comment_count, 
       source_branch, 
       target_branch,  
       vmr.versifit_email
       url, 
       assignees, 
       reviewers
from view_merge_requests vmr
inner join dim_sprints sp on sp.sprint_name = :sprint 
 left outer join employee_data_view edv on edv.email_address = vmr.author_email
where vmr.scrum_team = :team
and vmr.created_at between sp.start_date and sp.completion_date
; 

confluence updates:
select space, title, version ver_updated, conf.contributors, createdby, page_createdby author, page_createddate,  link, ded.scrum_team, conf.lastupdatedby
from VIEW_CONFLUENCE_CONTRIBUTORS conf 
left outer join employee_data_view ded on ded.email_address = conf.createdby
inner join view_dim_sprints vds on vds.sprint_name = :sprint
where conf.createddate between vds.start_date and vds.end_date
and ded.scrum_team = :team
;

issues related to sprint:
-- fact_jira_sprint_added_removed date removed is always >= date added 
select jv.issue_key
       ,summary
       ,issue_type
       ,epic_summary
       ,timestampdiff(day, start_date, date_added_utc) days_added
       ,case when fjs.date_added_utc <= timeadd(hour, 24, sp.start_date) and (date_removed_utc is null or date_removed_utc >= sp.completion_date) then 'planned'  -- given grace of 24 hours
          when fjs.date_added_utc <= timeadd(hour, 24, sp.start_date) and date_removed_utc <= sp.completion_date then 'planned and removed' 
          when fjs.date_added_utc > timeadd(hour, 24, sp.start_date) and (date_removed_utc is null or date_removed_utc >= sp.completion_date) then 'added late'
          when fjs.date_added_utc > timeadd(hour, 24, sp.start_date) and date_removed_utc <= sp.completion_date then 'added late and removed'
          else 'cannot determine' end planned_status
       ,case when jv.resolution_date between sp.start_date and sp.completion_date then 'completed within sprint'
          when jv.resolution_date > sp.completion_date then 'completed after sprint'
          when jv.resolution_date < sp.start_date then 'completed before sprint start'
          when jv.resolution_date is null then 'still open'
          else 'cannot determine' end completion_status
       ,devinprogressformated
       ,formated_cycle_time
       ,datediff(day, GREATEST(date_added_utc,sp.start_date), resolution_date) resolved_days_after       
       ,team       
       ,status_ordered
       ,array_to_string(fixversions,',')
       ,component_extn
       ,comment_with_git_commit
       ,merge_request_url
       ,sp.sprint_name
       ,sprints
       ,story_points
       ,issue_url
       ,assignee
       ,resolution_date
       ,start_date
       ,completion_date
       ,fjs.date_added_utc
       ,fjs.date_removed_utc
       ,jv.resolution_date
from jira_issues_view jv
inner join dim_sprints sp on sp.sprint_name = :sprint
inner join fact_jira_sprint_added_removed fjs on jv.issue_key = fjs.issue_key and fjs.sprint_name = sp.sprint_name
where --and array_contains(sp.sprint_name::variant,jv.sprints)
 fjs.date_added_utc is not null -- issue is added to sprint no matter when 
and (fjs.date_removed_utc is null or fjs.date_removed_utc >= timeadd(hour, 24, sp.start_date)) -- issue is not removed from sprint or its removed after start date with grace
and jv.team = :team
and jv.issue_type = :issuetype
;