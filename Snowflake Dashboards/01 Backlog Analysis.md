![[Pasted image 20250328162759.png]]

SELECT mnth , SUM(created_count) AS incoming, SUM(resolved_count) AS outgoing
FROM (
    SELECT resolved_month AS mnth, COUNT(*) AS resolved_count, 0 AS created_count
    FROM jira_issues_view_v2
    WHERE issue_type = 'Bug'
      AND department like '%Support%'
      and priority = :priority
      and sales_force_links is not null 
      and epic_url != 'https://powerschoolgroup.atlassian.net/browse/UIHN-2093' 
      and (resolution_date >= '2023-01-01')
      and issue_key like 'UIHN%'
    GROUP BY resolved_month
    UNION ALL
    SELECT created_month AS mnth, 0 AS resolved_count, COUNT(*) AS created_count
    FROM jira_issues_view_v2
    WHERE issue_type = 'Bug'
      AND department like '%Support%'
      and priority = :priority
      and sales_force_links is not null 
      and (created >= '2023-01-01')
      and epic_url != 'https://powerschoolgroup.atlassian.net/browse/UIHN-2093' 
      and issue_key like 'UIHN%'
    GROUP BY created_month
) subquery
where mnth >= '2023-01'
GROUP BY mnth;

SELECT mnth , SUM(created_count) AS incoming, SUM(resolved_count) AS outgoing
FROM (
    SELECT resolved_month AS mnth, COUNT(*) AS resolved_count, 0 AS created_count
    FROM jira_issues_view_v2
    WHERE issue_type = 'Bug'
      AND department like '%Cloud%' 
      and priority = :priority
      --and sales_force_links = :salesforceid
      and epic_url != 'https://powerschoolgroup.atlassian.net/browse/UIHN-2093' 
      and (resolution_date >= '2023-01-01')
      and issue_key like 'UIHN%'
    GROUP BY resolved_month
    UNION ALL
    SELECT created_month AS mnth, 0 AS resolved_count, COUNT(*) AS created_count
    FROM jira_issues_view_v2
    WHERE issue_type = 'Bug'
      AND department like '%Cloud%' 
      and priority = :priority
      --and sales_force_links = :salesforceid
      and (created >= '2023-01-01')
      and epic_url != 'https://powerschoolgroup.atlassian.net/browse/UIHN-2093' 
      and issue_key like 'UIHN%'
    GROUP BY created_month
) subquery
where mnth >= '2023-01'
GROUP BY mnth;

SELECT mnth , SUM(created_count) AS incoming, SUM(resolved_count) AS outgoing
FROM (
    SELECT resolved_month AS mnth, COUNT(*) AS resolved_count, 0 AS created_count
    FROM jira_issues_view_v2
    WHERE issue_type = 'Bug'
      AND department like '%Cloud%' 
      and priority = :priority
      --and sales_force_links = :salesforceid
      and epic_url != 'https://powerschoolgroup.atlassian.net/browse/UIHN-2093' 
      and (resolution_date >= '2023-01-01')
      and issue_key like 'UIHN%'
    GROUP BY resolved_month
    UNION ALL
    SELECT created_month AS mnth, 0 AS resolved_count, COUNT(*) AS created_count
    FROM jira_issues_view_v2
    WHERE issue_type = 'Bug'
      AND department like '%Cloud%' 
      and priority = :priority
      --and sales_force_links = :salesforceid
      and (created >= '2023-01-01')
      and epic_url != 'https://powerschoolgroup.atlassian.net/browse/UIHN-2093' 
      and issue_key like 'UIHN%'
    GROUP BY created_month
) subquery
where mnth >= '2023-01'
GROUP BY mnth;

incoming by components:
   SELECT created_month AS mnth, case when array_size(components) > 0 then components[0] else  component_extn end component,  COUNT(*) AS created_count
    FROM jira_issues_view
    WHERE issue_type = 'Bug'
      AND department = :department      
      and priority = :priority
      and sales_force_links = :salesforceid
      and epic_url != 'https://powerschoolgroup.atlassian.net/browse/UIHN-2093' 
      and (created >= '2023-01-01')
      and jira_project = :jiraproject     
      and component_extn = :component
      -- AND COMPONENTS[0] LIKE '%ETL%'
    GROUP BY created_month,component;

incoming by customers:
    SELECT created_month AS mnth, sf_customer_account_name,  COUNT(*) AS created_count
    FROM jira_issues_view_v2
    WHERE issue_type = 'Bug'
      AND department = :department      
      and priority = :priority
      and sales_force_links = :salesforceid
      and (created >= '2023-01-01')
      and jira_project = :jiraproject
      and epic_url != 'https://powerschoolgroup.atlassian.net/browse/UIHN-2093' 
    GROUP BY created_month, sf_customer_account_name;


issues closed in SF but open in jira
select issue_url, 
coalesce(sf_customer_name,sf_account_name) cust_or_account, 
status jira_status,
sf_status, 
--sf_date_openened,  
sf_date_closed,
age,
DATEDIFF(DAY, sf_date_closed, CURRENT_DATE()) AS days_since_closed_sf,
sales_force_links,
--sf_case_owner,
summary, 
team, creator
from jira_issues_view
where resolution_date is null
and issue_type = 'Bug'
and department = :department
and age_bucket = :age_bucket
and priority = :priority
and sales_force_links = :salesforceid
and sf_status = 'Closed'
order by days_since_closed_sf desc
;

cumulative backlog month end:
WITH created_months AS (
    SELECT DISTINCT 
        created_month,  
        TO_DATE(created_month, 'YYYY-MM') AS start_of_month, 
        LAST_DAY(TO_DATE(created_month, 'YYYY-MM')) AS end_of_month
    FROM jira_issues_view
    WHERE issue_type = 'Bug'
      AND department = :department      
      AND priority = :priority
      AND sales_force_links = :salesforceid      
), 
initial_backlog AS (
    SELECT COUNT(*) AS open_issues
    FROM jira_issues_view
    WHERE issue_type = 'Bug'
        AND department = :department      
        AND priority = :priority
        AND sales_force_links = :salesforceid
        AND jira_project = :jiraproject
        and epic_url != 'https://powerschoolgroup.atlassian.net/browse/UIHN-2093' 
        AND created <= '2023-04-30'  -- Bugs created on or before January 2023        
        AND (earliest_resolution_date IS NULL OR earliest_resolution_date > '2023-04-30')  
), 
issue_status AS (
    SELECT 
        cm.created_month AS mnth,  
        CASE WHEN jv.created <= cm.end_of_month THEN 1 ELSE 0 END AS incoming, 
        CASE WHEN jv.earliest_resolution_date <= cm.end_of_month THEN 1 ELSE 0 END AS outgoing 
    FROM jira_issues_view jv    
    FULL OUTER JOIN created_months cm 
    ON jv.created_month = cm.created_month
    WHERE jv.issue_type = 'Bug'
        AND jv.department = :department      
        AND jv.priority = :priority
        AND jv.sales_force_links = :salesforceid      
        AND jv.jira_project = :jiraproject     
        and jv.epic_url != 'https://powerschoolgroup.atlassian.net/browse/UIHN-2093' 
)
SELECT mnth, (COALESCE(ib.open_issues, 0) + SUM(istaus.incoming) - SUM(istaus.outgoing)) AS backlog
FROM issue_status istaus
LEFT JOIN initial_backlog ib 
WHERE mnth >= '2023-04'
GROUP BY mnth, ib.open_issues
ORDER BY mnth;

byteam by component: 
SELECT component_extn, team,
COUNT(*) AS ticket_count
FROM jira_issues_view
WHERE issue_type = 'Bug'
and resolution_date IS NULL
AND department = :department
and age_bucket = :age_bucket
and priority = :priority
and sales_force_links = :salesforceid
group by component_extn, team

By team by Age Bucket: 
SELECT age_bucket, team,
COUNT(*) AS ticket_count
FROM jira_issues_view
WHERE issue_type = 'Bug'
and resolution_date IS NULL
AND department = :department
and age_bucket = :age_bucket
and priority = :priority
and sales_force_links = :salesforceid
group by age_bucket, team;

Age 
SELECT age_bucket, team,
COUNT(*) AS ticket_count
FROM jira_issues_view
WHERE issue_type = 'Bug'
and resolution_date IS NULL
AND department = :department
and age_bucket = :age_bucket
and priority = :priority
and sales_force_links = :salesforceid
group by age_bucket, team
