# velocity
Track development velocity

Files in `BigQuery/*.sql` are Google BigQuery SQLs that generated data in `data/` directory

`analysis.rb` is a tool that process input file (which is an output from BigQuery) and generates final data for Bubble/Motion Google Sheet Chart.
It uses:
- hints file with additional repo name -> project mapping. (N repos --> 1 Project), so project name will be in many lines
- urls file which defines URLs for defined projects (separate file because in hints file we would have to duplicate data for each project ) (1 Project --> 1 URL)
- default map file which defines non standard names for projects generated automatically via groupping by org (like aspnet --> ASP.net) or to group multiple orgs and/or repos into single project. It is a last step of project name mapping

# Example use:
`ruby analysis.rb data/data_yyyymm.csv projects/projects_yyyymm.csv map/hints.csv map/urls.csv map/defmaps.csv skip.csv ranges.csv`

`input.csv` data/data_yyyymm.csv from BigQuery, like this:
```
org,repo,activity,comments,prs,commits,issues,authors
kubernetes,kubernetes/kubernetes,11243,9878,720,70,575,40
ethereum,ethereum/go-ethereum,10701,570,109,43,9979,14
...
```

`output.csv` to be imported via Google Sheet (File -> Import) and then chart created from this data. It looks like this:
```
org,repo,activity,comments,prs,commits,issues,authors,project,url
dotnet,corefx+coreclr+roslyn+cli+docs+core-setup+corefxlab+roslyn-project-system+sdk+corert+eShopOnContainers+core+buildtools,20586,14964,1956,1906,1760,418,dotnet,microsoft.com/net
kubernetes+kubernetes-incubator,kubernetes+kubernetes.github.io+test-infra+ingress+charts+service-catalog+helm+minikube+dashboard+bootkube+kargo+kube-aws+community+heapster,20249,15735,2013,1323,1178,423,Kubernetes,kubernetes.io
...
```

`hints.csv` CSV file with hints for repo --> project, it looks like this:
```
repo,project
Microsoft/TypeScript,Microsoft TypeScript
...
```

`urls.csv` CSV file with project --> url mapping
```
project,url
Angular,angular.io
...
```

`defmaps.csv` CSV file with better names for projects generated as default groupping within org:
```
name,project
aspnet,ASP.net
nixpkgs,NixOS
Azure,=SKIP
...
```
Special flag =SKIP  for project mean that this org should NOT be groupped

`skip.csv` CSV file that contain lists of repos and/or orgs and/or projects to be skipped in analysis:
```
org,repo,project
"enkidevs,csu2017sp314,thoughtbot,illacceptanything,RubySteps,RainbowEngineer",Microsoft/techcasestudies,"Apache (other),OpenStack (other)"
"2015firstcmsc100,swcarpentry,exercism,neveragaindottech,ituring","mozilla/learning.mozilla.org,Microsoft/HolographicAcademy,w3c/aria-practices,w3c/csswg-test",
"orgX,orgY","org1/repo1,org2/repo2","project1,project2"
```

`ranges.csv` CSV file containing ranges of repos properties that makes repo included in calculations.
It can constrain any of "commits, prs, comments, issues, authors" to be within range n1 .. n2 (if n1 or n2 < 0 then this value is skipped, so -1..-1 means unlimited
There can be also exception repos/orgs that do not use those ranges:
```
key,min,max,exceptions
activity,50,-1,"kubernetes,docker/containerd,coreos/rkt"
comments,20,100000,"kubernetes,docker/containerd,coreos/rkt"
prs,10,-1,"kubernetes,docker/containerd,coreos/rkt"
commits,10,-1,"kubernetes,kubernetes-incubator"
issues,10,-1,"kubernetes,docker/containerd,coreos/rkt"
authors,3,-1,"kubernetes,docker/containerd,google/go-github"
```

Generated output file contains all data from input (so it can be 600 rows for 1000 input rows for example).
You should manually review generated output and choose how much rows do You need.

`hintgen.rb` is a tool that takes data already processed for various created charts and creates distinct projects hint file from it:

`hintgen.rb data.csv map/hints.csv`
Use multiple times putting different files as 1st parameter (`data.csv`) and generate final `hints.csv`.

Already generated data:
- data/data_YYYYMM.csv --> data for given YYYYMM from BigQuery.
- projects/projects_YYYYMM.csv --> data generated by `analysis.rb` from data_YYYYMM.csv using: `map/`: `hints.csv`, `urls.csv`, `defmaps.csv`


`generate_motion.rb` tool is used to merge data from multiple files into one for motion chart. Usage:

`ruby generate_motion.rb projects/files.csv motion/motion.csv motion/motion_sums.csv [projects/summaries.csv]`

File `files.csv` contains list of data files to be merged, it looks like this:
```
name,label
projects/projects_201601.csv,01/2016
projects/projects_201602.csv,02/2016
...
```

Generates 2 output files:
- 1st is a motion data from each file with given label
- 2nd is cumulative sum of data, so 1st label contains data from 1st label, 2nd contains 1st+2nd, 3rd=1st+2nd+3rd ... last = sum of all data. Labels are summed in alphabetical order so if using months please use "YYYYMM" or "YYYY-MM" that will give correct results, and not "MM/YYYY" that will for example swap "2/2016" and "1/2017"

Output formats of 1st and 2nd files are identical.

First column is data file generated by `analysis.rb` another column is label that will be used as "time" for google sheets motion chart
Output is in format:
```
project,url,label,activity,comments,prs,commits,issues,authors,sum_activity,sum_comments,sum_prs,sum_commits,sum_issues,sum_authors
Kubernetes,kubernetes.io,2016-01,6289,5211,548,199,331,73,174254,136104,18264,8388,11498,373
Kubernetes,kubernetes.io,2016-02,13021,10620,1180,360,861,73,174254,136104,18264,8388,11498,373
...
Kubernetes,kubernetes.io,2017-04,174254,136104,18264,8388,11498,373,174254,136104,18264,8388,11498,373
dotnet,microsoft.com/net,2016-01,8190,5933,779,760,718,158,158624,111553,17019,17221,12831,382
dotnet,microsoft.com/net,2016-02,17975,12876,1652,1908,1539,172,158624,111553,17019,17221,12831,382
...
dotnet,microsoft.com/net,2017-04,158624,111553,17019,17221,12831,382,158624,111553,17019,17221,12831,382
VS Code,code.visualstudio.com,2016-01,7526,5278,381,804,1063,112,155621,104386,9501,17650,24084,198
VS Code,code.visualstudio.com,2016-02,17139,11638,986,1899,2616,133,155621,104386,9501,17650,24084,198
...
VS Code,code.visualstudio.com,2017-04,155621,104386,9501,17650,24084,198,155621,104386,9501,17650,24084,198
...
```
Each row contains its label data (cumulative or separata) and columns with staring with `max_` conatin cumulative data for all labels.
This is to make this data easy available for google sheet motion chart without complex cell indexing.

Final optional file `summarier.csv` can be used to read number of authors from it. This is because number of authors is computed in a different way.
Without summaries file (or if given project is not in summaries file) we have number of distinct authors in each period. To get summary value for all periods we're just getting max of all periods.
This is obviously not a real count of all distinct authors in all periods. So if we give another file which contains summary data for one big period that is euual to sum of all periods - then we can get number of authors from there.

To manually add other projects (like Linux) use `add_linux.sh` or create similar tools for other projects. Data for was generated manually using custom `gitdm` tool (`github cncf/gitdm`) on `torvalds/linux` repo and via manually counting emails in different periods on LKML.
Example usage (assuming Linux additional data in `data/data_linux.csv), could be: 
`ruby add_linux.rb data/data_201603.csv data/data_linux.csv 2016-03-01 2016-04-01`

To get some repos data from some file and put it in some other file use:
`ruby merger.rb file_to_merge.rb file_to_get_data_from.rb`
See for example `./shells/top30_201605_201704.sh`

To process "unlimited" data from BigQuery output (file `data/unlimited.csv`) please use `shells/unlimited.sh` or `shells/unlimited_both.sh`).
Unlimited means that BigQuery is not constraining repositories by having commits, comments, issues, PRs, authors > N (this N is 5-50 depending on which metric: authors for example is 5 while comments is 50).
Unlimited only requires that authors, comments, commits, prs, issues are all > 0.
And then only CSV `map/ranges_unlimited.csv` is used to further constrain data. This basically moves filtering out of BigQuery (so it can be called once) to Ruby tool.
And `shells/unlimited_both.sh` uses `map/ranges_unlimited.csv` that is not setting ANY limit:
```
key,min,max,exceptions
activity,-1,-1,
comments,-1,-1,
prs,-1,-1,
commits,-1,-1,
issues,-1,-1,
authors,-1,-1,
```
It means that mapping must have extermally long list of projects from repos/orgs to get real non obfuscated data.

You can skip ton of organization's small repos (if they not sum up to just few projects, but all all distinct), with:
`rauth[res[res.map { |i| i[0] }.index('Google')][0]].select { |i| i.split(',')[1].to_i < 14 }.map { |i| i.split(',')[0] }.join(',')`
This is an example on Google.
Say Top 100 porjects have 100th project with 290 authors.
All tiny google repos (distinct small projects) will sum up and make Google overall 15th (for example).
This command generates output list of google repos with < 14 authors. You can put results in map/skip.csv" and then You'll avoid false positive top 15 for Google overall (which means nothing)

There is also tool to add data for external projects (not hosted on GitHub): `add_external.rb`.
It is used by `shells/unlimited.csv` and `shells/unlimited_both.sh`
Example call:
`ruby add_external.rb data/unlimited.csv data/data_gitlab.csv 2016-05-01 2017-05-01 gitlab gitlab/GitLab`
It requires CSV file with external repo data.
It must be defined per date range.
It has format (for example `data/data_gitlab.csv`:
```
org,repo,from,to,activity,comments,prs,commits,issues,authors
gitlab,gitlab/GitLab,2016-05-01,2017-05-01,40000,40000,11595,9479,22821,1500

```

There is also a tool to update generated projects file (that is used to import data for chart).
`update_projects.rb`
Used in `shells/unlimited_both.sh`
It is used to update cerain values in given projects
It uses input file with format:
```
project,key,value
Apache Mesos,issues,7581
Apache Spark,issues,5465
Apache Kafka,issues,1496
Apache Camel,issues,1284
Apache Flink,issues,2566
Apache (other),issues,52578
```
This allows to update specific keys in specific projects with data taken from other source than GitHub.
It is used now to update github data with issues statistics from jira (for apache projects).


Tool to create per project ranks (for all project's numeric properties) `report_projects_ranks.rb` & `shells/report_project_ranks.sh`
Shell script projects from `projects/unlimited_both.csv` and uses: `reports/cncf_projects_statistics.csv` file to get a list of projects that needs to be included in rank statistics.
File format is:
```
project
project1
project2
...
projectN
```
Output rank statistics file is `reports/cncf_projects_ranks.txt`

There are also special cases (see `./shells/unlimited_both.sh` that call all cases in correct order)
Some details about external data used to add non GitHub projects:
- How to find Apache issues in Jira: `res/data_apache_jira.query`

- Chromium case: (details here: `res/data_chromium_bugtracker.txt`), issues from their bugtracker, number of authors and commits in date range via `git log` oneliner:
Must be called in Git repo cloned from GoogleSource (not from github), here: `git clone https://chromium.googlesource.com/chromium/src`
Commits: `git log --since "2016-05-01" --until "2017-05-01" --pretty=format:"%H" | sort | uniq | wc -l` gives 77437
Authors: `git log --since "2016-05-01" --until "2017-05-01" --pretty=format:"%aE" | sort | uniq | wc -l` gives 1663
To analyse those commits (and exclude merge and robot commits):
data/data_chromium_commits.csv, run while in chromium/src repository:
`git log --since "2016-05-01" --until "2017-05-01" --pretty=format:"%aE~~~~%aN~~~~%H~~~~%s" | sort | uniq > chromium_commits.csv`
Then remove special CSV characters with VI commands: `:%s/"//g`, `:%s/,//g`
Then add CSV header manually "email,name,hash,subject" and move it to: `data/data_chromium_commits.csv`
Finally replace '~~~~' with ',' to create correct CSV: `:%s/\~\~\~\~/,/g`
Then run `ruby commits_analysis.rb data/data_chromium_commits.csv map/skip_commits.csv` or `./shells/chromium_commits_analysis.sh`

- OpenStack case here: `res/data_openstack_lanuchpad.query` data from their launchpad

- WebKit case here: `res/data_webkit_links.txt` issues from their bugtracker: `https://webkit.org/reporting-bugs/`
For authors and commits tried 3 different tools: our cncf/gitdm on their webkit/WebKit github repo, git one liner on the same repo (`git clone git://git.webkit.org/WebKit.git WebKit`):
Authors: 121: `git log --since "2016-05-01" --until "2017-05-01" --pretty=format:"%aE" | sort | uniq | wc -l`
Authors: 121: `git log --since "2016-05-01" --until "2017-05-01" --pretty=format:"%cE" | sort | uniq | wc -l`
Commits: 13051: `git log --since "2016-05-01" --until "2017-05-01" --pretty=format:"%H" | sort | uniq | wc -l`
Our cncf/gitdm output files are also stored here: `res/webkit/`: WebKit_2016-05-01_2017-05-01.csv  WebKit_2016-05-01_2017-05-01.txt

And also tried SVN one liner on their original SVN repo (Github is only a mirror): 
To fetch SVN repo:
`svn checkout https://svn.webkit.org/repository/webkit/trunk WebKit`
or:
`tar jxvf WebKit-SVN-source.tar.bz2`
`cd webkit`
`svn switch --relocate http://svn.webkit.org/repository/webkit/trunk https://svn.webkit.org/repository/webkit/trunk`
And then run their script: `update-webkit`

Number of commits: svn log -q -r {2016-05-01}:{2017-05-01} | sed '/^-/ d' | cut -f 1 -d "|" | sort | uniq | wc -l
Number of authors: svn log -q -r {2016-05-01}:{2017-05-01} | sed '/^-/ d' | cut -f 2 -d "|" | sort | uniq | wc -l
To See data from SVN:
Revisions: svn log -q -r {2017-05-25}:{2017-05-26} | sed '/^-/ d' | cut -f 1 -d "|"
Authors: svn log -q -r {2017-05-25}:{2017-05-26} | sed '/^-/ d' | cut -f 2 -d "|"
Dates: svn log -q -r {2017-05-25}:{2017-05-26} | sed '/^-/ d' | cut -f 3 -d "|"

- GitLab estimation and details here: `res/gitlab_estims.txt`
- LibreOffice case: see `res/libreoffice_git_repo.txt`

To add new non-standard project (but from github mirros which can have 0s on comments, commits, issues, prs, activity, authors) follow this route:
- Copy `BigQuery/org_finder.sql` to clipboard and run this on BigQuery replacing condition for org (for example lower(org.login) like '%your%org%)
- Examine output org/repos combination (manually on GitHub) and decide about final condition for run final BigQuery
- Copy `BigQuery/query_apache_projects.sql` into some `BigQuery/query_your_project.sql` then update conditions to those found in the previous step
- Run the query
- Save results to table, then export table to GStorage, then download this table as CSV from GStorage into `data/data_your_project_datefrom_date_to.csv`
- Add this to `shells/unlimited_both.csv`:
```
echo "Adding/Updating YourProject case"
ruby merger.rb data/unlimited.csv data/data_your_project_datefrom_date_to.csv
```
- Update `map/range*.csv` - add exception for YourProject (because it can have 0s now - this is output from BigQuery without numeric conditions)
- Run `shells/unlimited_both.sh` and examine Your Project (few iterations to add correct mapping in `./map/`: hints, defmaps, urls etc.)
- You can run manually: `ruby analysis.rb data/unlimited.csv projects/unlimited_both.csv map/hints.csv map/urls.csv map/defmaps.csv map/skip.csv map/ranges_sane.csv`
- For example see YourProject rank: `res.map { |i| i[0] }.index('LibreOffice')` or `res[res.map { |i| i[0] }.index('LibreOffice')][2][:sum]`
- SOme of the values will be missing (like for example PRs for mirror repos)
- Now comes non standard path, please see `shells/unlimited_both.sh` for non standar data update that comes after final `ruby analysis.rb` call - this is usually different for each non-standard project

# Most Up to date process
To generate all data for Top 40 chart: https://docs.google.com/spreadsheets/d/1hD-hXlVT60AGhGVifNn7nNo9oVMKnIoQ2kBNmx-YY8M/edit?usp=sharing

- Fetch all data needed using BigQuery (once - or use data already fetched present in this repo).
- If fetched new BigQuery data then rerun special projects BigQuery analysis scripts: ./shells/: run_apache.sh, run_chrome_chromium.sh, run_cncf.sh, run_openstack.sh
- To just regenerate all other data: run `./shells/unlimited_both.sh`
- See per project ranks statistics: `reports/cncf_projects_ranks.txt
- Get final output file `projects/unlimited.csv` and import it on A50 cell in `https://docs.google.com/spreadsheets/d/1hD-hXlVT60AGhGVifNn7nNo9oVMKnIoQ2kBNmx-YY8M/edit?usp=sharing` chart


# Example - generate chart for new data range
We aleary have `shells/unlimited_both.sh` that generates our chart for 2016-05-01 to 2017-05-01. We want to generate next chart for new date range: 2016-06-01 to 2017-06-01.
This is a step by step tutorial how to do it.
- Copy `shells/unlimited_both.sh` to `shells/unlimited_20160601-20170601.sh`
- Have `shells/unlimited_20160601-20170601.sh` opened in some other terminal window `vi shells/unlimited_20160601-20170601.sh` and we need to update all steps
- First we need unlimited BigQuery output for a new date range:
```
echo "Restoring BigQuery output"
cp data/unlimited_output_201605_201704.csv data/unlimited.csv
```
- We need `data/unlimited_output_201606_201705.csv` file. To generate this one we need to run BigQuery for new date range.
- Open file that generated current range: `vi BigQuery/query_201605_201704_unlimited.sql`
- Save it as: `BigQuery/query_201606_201705_unlimited.sql` while changing date ranges in SQL.
- Copy it to clipboard `pbcopy < BigQuery/query_201606_201705_unlimited.sql` and run in Google BigQuery: `https://bigquery.cloud.google.com/queries/<<your_google_project_name>>`
- Save result to table `<<your_google_user_name>>:unlimited_201606_201705`, it takes about 1TB and costs about $5 "Save as table"
- Open this table `<<your_google_user_name>>:unlimited_201606_201705` and click "Export Table" and export it to google storage as: `gs://<<your_google_user_name>>/unlimited_201606_201705.csv` (You can clisk "View files" to see files in Your gstorage)
- Go to google storage and download `<<your_google_user_name>>/unlimited_201606_201705.csv` and put it where `shells/unlimited_20160601-20170601.sh` expects it (update file name to `data/unlimited_output_201606_201705.csv`): 
```
echo "Restoring BigQuery output"
cp data/unlimited_output_201606_201705.csv data/unlimited.csv
```
- Now we have main data (step 1) finished for new chart, now we need to get data for all non-standard projects. You can try our analysis tool without any special projects by running:
`ruby analysis.rb data/unlimited.csv projects/unlimited_both.csv map/hints.csv map/urls.csv map/defmaps.csv map/skip.csv map/ranges_sane.csv`
- There can be some new projects that are unknown, ranks can chage, so there can be manual changes needed to mappings in `map/` directory: `hints.csv`, `defmaps.csv` and `urls.csv`. Maybe also `skip.csv` (if there are new projects that should be skipped)
- This is what I've got on 1st run:
```
Project #23 (org, 457) skillcrush (skillcrush) (skillcrush-104) have no URL defined
Project #45 (org, 366) pivotal-cf (pivotal-cf) (...) have no URL defined
Project #50 (org, 353) Automattic (Automattic) (...) have no URL defined
```
- Let's what are top authors projects for those non-found projects: `rauth[res[res.map { |i| i[0] }.index('Automattic')][0]]`
- Then we must add entriues for few top ones in `map/hints.csv` say with >= 20 authors:
```
Automattic/amp-wp,31
Automattic/wp-super-cache,29
Automattic/simplenote-electron,22
Automattic/happychat-service,21
Automattic/kue,20
```
We need to examine each on `github.com`, like for 1st: `github.com/Automattic/amp-wp`. We see that this is a WordPress plugin, so belnog to wWrdpress/WP Calypso project:
`grep -HIn "wordpress" map/*.csv`
`grep -HIn "WP Calypso" map/*.csv`
We see that we have WP Calypso defined in hints file:
```
map/hints.csv:23:Automattic/WP-Job-Manager,WP Calypso
map/hints.csv:24:Automattic/facebook-instant-articles-wp,WP Calypso
map/hints.csv:26:Automattic/sensei,WP Calypso
map/hints.csv:29:Automattic/wp-calypso,WP Calypso
map/hints.csv:30:Automattic/wp-e2e-tests,WP Calypso
map/urls.csv:438:WP Calypso,developer.wordpress.com/calypso
```
Just add new repo for this project (`map/hints.csv`), row: `Automattic/amp-wp,WP Calypso`
Do the same for other projects/repos. Re-run analysis tool untill all is fine.
- For example after definiing some new projects we see "EPFL-SV-cpp-projects" as one of top 50. This is an educational org that should be skipped. Add it to `map/skip.csv` for skipping row: `EPFL-SV-cpp-projects,,`
- Once You have all URL's defined, added new mapping You can see preview of Top projects on while stopped in `binding.pry`, by typing `all`. Now we need to go back to `shells/unlimited_20160601-20170601.sh` and regenerate all non standard data (for projects not on github or requiring special queries on github - for example because of having 0 activity, comments, commits, issues, prs or authors)

- Now Linux case: we need to change line `ruby add_linux.rb data/unlimited.csv data/data_linux.csv 2016-05-01 2017-05-01` into `ruby add_linux.rb data/unlimited.csv data/data_linux.csv 2016-06-01 2017-06-01` and run it
- You will see: `Data range not found in data/data_linux.csv: 2016-06-01 - 2017-06-01` that meens we need to add new data range for Linux in file: `data/data_linux.csv`
- Data for linux is here `https://docs.google.com/spreadsheets/d/1CsdreHox8ev89WoP6LjcryroKDOH2gQipMC9oS95Zhc/edit?usp=sharing` but it doesn have May 2017 (finished yesterday), so we need last month's data.
- Go to: `https://lkml.org/lkml/2017` and copy May 2017 into linked google spreadsheet: (22110).
- Add row for May 2017 to `data/data_linux.csv`: `torvalds,torvalds/linux,2017-05-01,2017-06-01,0,0,0,0,22110` - You can see that we only have "emails" column now. Other columns must be fetech from linux kernel repo using `cncf/gitdm` analysis:
- You can also sum issues from the sheet to get 2016-06-01 - 2017-06-01: (254893): `torvalds,torvalds/linux,2016-06-01,2017-06-01,0,0,0,0,254893`
- Now `cncf/gitdm` on linux kernel repo: `cd ~/dev/linux && git checkout master && git reset --hard && git pull`. ALternative is (if You don't have linux repo cloned): `cd ~/dev/`, `git clone https://github.com/torvalds/linux.git`.
- Go to `cncf/gitdm`: `cd ~/dev/cncf/gitdm`, run: `./linux_range.sh 2017-05-01 2017-06-01`
- While on `cncf/gitdm`, see: `vim linux_stats/range_2017-05-01_2017-06-01.txt`:
```
Processed 1219 csets from 424 developers
34 employers found
A total of 24970 lines added, 14469 removed (delta 10501)
```
- You have values for `changesets,additions,removals,authors` here, update `cncf/velocity/data/data_linux.csv` accordingly.
- Do the same for `./linux_range.sh 2016-06-01 2017-06-01` and `linux_stats/range_2016-06-01_2017-06-01.txt`, Results:
```
Processed 64482 csets from 3803 developers
91 employers found
A total of 3790914 lines added, 1522111 removed (delta 2268803)
```
- Final linux rows (one for May 2017, and another for last year including May 2017) are:
```
torvalds,torvalds/linux,2017-05-01,2017-06-01,1219,24970,14469,424,22110
torvalds,torvalds/linux,2016-06-01,2017-06-01,64482,3790914,1522111,3803,254893
```
- NOTE that those numbers are lower than usual (generated June 1st), maybe torvalds/linux mirror wasn't fully updated yet? Issues from LKMA are little higher than in April, so wtf? TODO: check this again after 5th June!

- GitLab case: Their repo is: `https://gitlab.com/gitlab-org/gitlab-ce/`, clone it via: `git clone https://gitlab.com/gitlab-org/gitlab-ce.git` in `~/dev/` directory.
- Their repo hosted by GitHub is: `https://github.com/gitlabhq/gitlabhq`, clone it via `git clone https://gitlab.com/gitlab-org/gitlab-ce.git` in `~/dev/` directory.
- Go to `cncf/gitdm` and run GitLab repo analysis: `./repo_in_range.sh ~/dev/gitlab-ce/ gitlab 2016-06-01 2017-06-01`
- Results `vim other_repos/gitlab_2016-06-01_2017-06-01.txt`:
```
Processed 16574 csets from 513 developers
15 employers found
A total of 926818 lines added, 548205 removed (delta 378613)
```
- Their bug tracker is `https://gitlab.com/gitlab-org/gitlab-ce/issues`, just count issues in given date range. Sort by "Last created" and count issues in given range:
There are 732 pages of issues (20 on page) = 14640 issues (`https://gitlab.com/gitlab-org/gitlab-ce/issues?page=732&scope=all&sort=created_desc&state=all`)
- To count Merge Requests (PRs): `https://gitlab.com/gitlab-org/gitlab-ce/merge_requests?scope=all&state=all`
Merge Requests: 371,5 page * 20 = 7430
- To count authors run in gitlab-ce directory: `git log --since "2016-06-01" --until "2017-06-01" --pretty=format:"%aE" | sort | uniq | wc -l` --> 575
- To count authors run in gitlab-ce directory: `git log --since "2016-05-01" --until "2017-05-01" --pretty=format:"%aE" | sort | uniq | wc -l` --> 589

- Cloud Foundry case:
- Copy: `BigQuery/query_cloudfoundry_201605_201704.sql` to `BigQuery/query_cloudfoundry_201606_201705.sql` and update conditions. Then run query on BigQuery console (see details at the beginning of example)
- Finally You will have `data/data_cloudfoundry_201606_201705.csv` (run query, save results to table, export table to gstorage, download csv from gstorage).
- Update (and eventually manually run) CF case (in `shells/unlimited_20160601-20170701.sh`): `ruby merger.rb data/unlimited.csv data/data_cloudfoundry_201606_201705.csv force`

- CNCF Projects case
- We have a line `ruby merger.rb data/unlimited.csv data/data_cncf_projects.csv` need to change to `ruby merger.rb data/unlimited.csv data/data_cncf_projects_201606_201705.csv`
- Copy: `cp BigQuery/query_cncf_projects.sql BigQuery/query_cncf_projects_201606_201705.sql`, update conditions: `BigQuery/query_cncf_projects_201606_201705.sql`
- Run on BigQuery and do the same as for CF case, final saved file will be: `data/data_cncf_projects_201606_201705.csv`
- Final line should be (try it): `ruby merger.rb data/unlimited.csv data/data_cncf_projects_201606_201705.csv`

- WebKit case
- Change merger line to `ruby merger.rb data/unlimited.csv data/webkit_201606_201705.csv`
- WebKit have no usable data on GitHub, so running BigQuery is not needed, we no longer need those lines for WebKit (we will just update `data/webkit_201606_201705.csv` file), remove them from current shell `shells/unlimited_20160601-20170601.sh`:
```
echo "Updating WebKit project using gitdm and other"
ruby update_projects.rb projects/unlimited_both.csv data/data_webkit_gitdm_and_others.csv -1
```
- Now we need to generate values for `data/webkit_201606_201705.csv` file:
- Issues:
Go to: https://webkit.org/reporting-bugs/
Search all bugs in webkit, order by modified desc - will be truncated to 10,000.
https://bugs.webkit.org/buglist.cgi?bug_status=UNCONFIRMED&bug_status=NEW&bug_status=ASSIGNED&bug_status=REOPENED&bug_status=RESOLVED&bug_status=VERIFIED&bug_status=CLOSED&limit=0&order=changeddate%20DESC%2Cbug_status%2Cpriority%2Cassigned_to%2Cbug_id&product=WebKit&query_format=advanced&resolution=---&resolution=FIXED&resolution=INVALID&resolution=WONTFIX&resolution=LATER&resolution=REMIND&resolution=DUPLICATE&resolution=WORKSFORME&resolution=MOVED&resolution=CONFIGURATION%20CHANGED
2016-12-13 --> 2017-06-01 = 9988 issues:
ruby> Date.parse('2017-06-01') - Date.parse('2016-12-13') => (170/1), (9988.0 * 365.0/170.0) --> 21444 issues
See how many days makes 10k, and estimate for 365 days (1 year): gives 22k bugs/issues
- Commits, Authors:
`cd ~dev/ && git clone git://git.webkit.org/WebKit.git WebKit`
xxx

- OpenStack case:
- Change line `ruby merger.rb data/unlimited.csv data/data_openstack_201605_201704.csv` to `ruby merger.rb data/unlimited.csv data/data_openstack_201606_201705.csv`
- To get `data/data_openstack_201606_201705.csv` file from BigQuery do:
- Copy `cp BigQuery/query_openstack_projects.sql BigQuery/query_openstack_projects_201606_201705.sql` and update date range condition in `BigQuery/query_openstack_projects_201606_201705.sql`
- Copy to clipboard `pbcopy < BigQuery/query_openstack_projects_201606_201705.sql` and run BigQuery, Save as Table, export to gstorage, and save result as `data/data_openstack_201606_201705.csv`
- Run `ruby merger.rb data/unlimited.csv data/data_openstack_201606_201705.csv` to try it

- Apache case:
- Exactly the same BigQuery steps as OpenStack for example, final line should be `ruby merger.rb data/unlimited.csv data/data_apache_201606_201705.csv`
- `cp BigQuery/query_apache_projects.sql BigQuery/query_apache_projects_201606_201705.sql`, update conditions, run BigQ, download results to `data/data_apache_201606_201705.csv`
- Run `ruby merger.rb data/unlimited.csv data/data_apache_201606_201705.csv`

- Chromium case
- Beginning (BigQuery part) exactly the same as Apache or OpenStack (just replace with word chromium): `ruby merger.rb data/unlimited.csv data/data_chromium_201606_201705.csv`

- openSUSE case
- Beginning (BigQuery part) exactly the same as Apache or OpenStack (just replace with word opensuse): `ruby merger.rb data/unlimited.csv data/data_opensuse_201606_201705.csv`

- LibreOffice case
- Beginning (BigQuery part) exactly the same as Apache or OpenStack (just replace with word libreoffice): `ruby merger.rb data/unlimited.csv data/data_libreoffice_201606_201705.csv`

(...)

- Finally `./projects/unlimited.csv` is generated. You need to import it in final Google chart by doing:
- Select A50 cell. Use File --> Import, then "Upload" tab, "Select a file from your computer", choose `./projects/unlimited.csv`
- Then "Import action" --> "replace data starting at selected call", click Import.
- Voila!
Final version will live here: https://docs.google.com/spreadsheets/d/1a2VdKfAI1g9ZyWL09TnJ-snOpi4BC9kaEVmB7IufY7g/edit?usp=sharing
But now it is WIP!

# Results:

NOTE: for viewing using those motion charts You'll need Adobe Flash enabled when clicking links. It works (tested) on Chrome and Safari with Adobe Flash installed and enabled.

For data from files.csv (data/data_YYYYMM.csv), 201601 --> 201703 (15 months)
Chart with cumulative data (each month is sum of this month and previous months) is here:
https://docs.google.com/spreadsheets/d/11qfS97WRwFqNnArRmpQzCZG_omvZRj_y-MNo5oWeULs/edit?usp=sharing
Chart with monthly data (that looks wrong IMHO due to google motion chart data interpolation between months) is here: 
https://docs.google.com/spreadsheets/d/1ZgdIuMxxcyt8fo7xI1rMeFNNx9wx0AxS-2a58NlHtGc/edit?usp=sharing

I suggest play around with 1st chart (cumulative sum):
It is not able to remember settings so once You click on "Chart1" scheet I suggest:
- Change axis-x and axis-y from Lin (linerar) to Log (logarithmics)
- You can choose what column should be used for color: I suggest activity (this is default and shows which project was most active) or choose unique color (You can select from commits, prs+issues, size) (size is square root of number of authors)
- Change playback speed (control next to play) to slowest
- Select inerested projects from Legend (like Kubernetes for example or Kubernetes vs dotnet etc) and check "trails"
- You can also change what x and y axisis use as data, defaults are: x=commits, y=pr+issues, and change scale type lin/log
- You can also change which column use for bubble size (default is "size" which means square root of number of authors), note that number of authors = max from all monts (distinct authors that contributed to activity), this is obviously differnt from set of distinct authors activity in entire 15 months range

On the top/right just above the Color drop down You can see additional two chart types:
- Bar chart - this can be very useful
- Choose li or log y-axis scale, then select Kubernetes from Legend and then choose any of y-axis possible values (activity, commits, PRs+issues, Size) and click play to see how Kubernetes overtakes multiple projects during our period.
Finally there is also a linear chart, take a look on it too.

# CNCF Projects
To generate data for CNCF projects:
- Run `BigGuery/query_cncf_projects.sql` on Google BigQuery Console. It takes about 800 GiB which is cost is about $4.
- Save output to GoogleSheets and download is as CSV file and save in `data/data_cncf_projects.csv` (File -> Download As -> Comma separated values ...)
- Process BigQuery output with velocity's analysis tool: `ruby analysis.rb data/data_cncf_projects.csv projects/projects_cncf.csv map/hints.csv map/urls.csv map/defmaps.csv` or use `shells/run_cncf.sh` which does the same
- Import output file `projects/projects_cncf.csv` as Google chart's data.

There is also gist here (but above description is more up to date): https://gist.github.com/lukaszgryglicki/093ced06455a3f14f0e4d25459525207

Links to various charts and videos generated using this project are here: `res/links.txt`

