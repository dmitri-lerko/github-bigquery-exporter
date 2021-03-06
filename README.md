# github-bigquery-exporter

GitHub BigQuery export utility, for those times when more granular PR and Issue queries are required. This is also good way to query data for periods longer than the GitHub max of `30` days.

## Requirements

### token

You can run the export script without the GitHub API token but you will be subject to much stricter rate limits. To avoid this (important for larger organizations) get a personal API tokens by following [these instructions](https://blog.github.com/2013-05-16-personal-api-tokens/) and define it in an `GITHUB_ACCESS_TOKEN` environment variable

```shell
export GITHUB_ACCESS_TOKEN="your long token string goes in here"
```

> Remember, you have to be org admin for this to work

### json2csv

GitHub API exports data in JSON format. The simplest way to import desired data elements is to convert the data into CSV using `json2csv`, a Node.js utility that converts JSON to CSV.

```shell
npm install -g json2csv
```

## Configuration

To configure the export script you will need to define the `organization` and provide list of `repositories` in this organization.

```shell
declare -r org="my-org-name"
declare -a repos=("my-repo-1"
                  "my-repo-2"
                  "my-repo-3")
```

Optionally, to configure the import script you can edit the data-set name and configure the `issue` and `pull` table name. This step is only required if you for some reason have name conflicts in your BiqQuery project.

```shell
declare -r ds="github"
declare -r issues_table="issues"
declare -r pulls_table="pulls"
```

## Export

To execute the GitHub export script run this command:

```shell
./export
```

The expected output should look something like this

```shell
Downloading issues for org/repo-1...
Downloading prs for org/repo-1...
Downloading issues for org/repo-2...
Downloading prs for org/repo-2...
```

## Import

To execute the BigQuery import script run this command:

```shell
./import
```

The expected output should look something like this

```shell
Dataset 'project:github' successfully created.
Table 'project:github.issues' successfully created.
Table 'project:github.pulls' successfully created.
Waiting on bqjob_..._1 ... (0s) Current status: DONE
Waiting on bqjob_..._1 ... (0s) Current status: DONE
```

### Query

When the above scripts completed successfully you should be able to query the imported data using SQL in BigQuery console. For example to find repositories with most issues over last 90 days:

```sql
select
  i.repo,
  count(*) num_of_issues
from gh.pulls i
where date_diff(CURRENT_DATE(), date(i.ts), day) < 90
group by
  i.repo
order by 2 desc
```

### TODO

* Add org user export/import
* Sort out the 2nd run where tables have to be appended
* Bash, really? Can I haz me a service?


## Scratch

### Users who have activity (pr/issue) but are NOT in the user table

```sql
with active_users as (
  select username
  from gh.issues
  group by username

  union all

  select p.username
  from gh.pulls p
  group by username
)
select *
from active_users
where username not in (SELECT username from gh.users)
```

> Export results as CSV and use them as input in `user-export` which will download the GitHUb data for each one of those users. Then, when done, run `user-import` to bring those users into

### Activity breakdown by company

```sql
select all_prs.company, all_prs.prs apr, coalesce(m3_prs.prs,0) rpr from (

  select
    COALESCE(u.company, 'Unknown') company,
    COUNT(*) prs
  from gh.pulls i
  join gh.users u on i.username = u.username
  group by company

) all_prs

left join (

  select
    COALESCE(u.company, 'Unknown') company,
    COUNT(*) prs
  from gh.pulls i
  join gh.users u on i.username = u.username
  where i.ts > "2018-10-30 23:59:59"
  group by company

) m3_prs on all_prs.company = m3_prs.company

order by 2 desc
```


```sql
select u.company, count(*)
from gh.pulls i join gh.users u on i.username = u.username
where u.company is not null
group by company order by 2 desc
```


```sql
select
  pr_month,
  sum(google_prs) as total_google_prs,
  sum(non_google_prs) as total_non_google_prs
from (
select
  case when u.company = 'Google' then 1 else 0 end as google_prs,
  case when u.company = 'Google' then 0 else 1 end as non_google_prs,
  TIMESTAMP_TRUNC(i.`on`, MONTH) as pr_month
from gh.pulls i
join gh.users u on i.username = u.username
where u.company  is not null
)
group by pr_month
order by 1
```

### PRs

```sql
select
  pr_month,
  sum(google_prs) as total_google_prs,
  sum(non_google_prs) as total_non_google_prs
from (
select
  case when u.company = 'Google' then 1 else 0 end as google_prs,
  case when u.company = 'Google' then 0 else 1 end as non_google_prs,
  TIMESTAMP_TRUNC(i.ts, MONTH) as pr_month
from gh.pulls p
join gh.users u on p.username = u.username
where u.company <> ''
)
group by pr_month
order by 1
```

### Issues

```sql
select
  pr_month,
  sum(google_prs) as total_google_prs,
  sum(non_google_prs) as total_non_google_prs
from (
select
  case when u.company = 'Google' then 1 else 0 end as google_prs,
  case when u.company = 'Google' then 0 else 1 end as non_google_prs,
  TIMESTAMP_TRUNC(i.ts, MONTH) as pr_month
from gh.issues i
join gh.users u on i.username = u.username
where u.company <> ''
)
group by pr_month
order by 1
```


```sql
select pr_month, repo, count(*) as prs
from (
select
  i.repo,
  TIMESTAMP_TRUNC(i.ts, MONTH) as pr_month
from gh.pulls i
join gh.users u on i.username = u.username
where u.company is not null
)
group by pr_month, repo
order by 1, 3 desc
```


```sql
select
  pr_month,
  repo,
  count(*) action
from (

  select
    repo,
    SUBSTR(CAST(TIMESTAMP_TRUNC(ts, MONTH) as STRING),0,7) as pr_month
  from gh.issues

  union all

  select
    repo,
    SUBSTR(CAST(TIMESTAMP_TRUNC(ts, MONTH) as STRING),0,7) as pr_month
  from gh.pulls

)
where repo = 'build' --'build-pipeline'
group by repo, pr_month
order by 1, 2
```

```sql
 select repo, count(*) from (
 select
    repo
  from gh.issues

  union all

  select
    repo
  from gh.pulls
)
group by repo
order by 2 desc
```