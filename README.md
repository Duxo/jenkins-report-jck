# jenkins-report-jck
Jenkins plugin to show unit-test, tesng, jtreg and JCK reports summaries, diffs and details

The plugin reads archived gzipped xml files prdoduced by junit/testng/jtreg/jck suites  ([or anyhow else generated](https://github.com/judovana/OpenJdkBuilder/blob/master/tck/autoruns/jtreg-shell-xml.sh)) runs.

* [Implementation details](#implementation-details)
* [Job run details](#job-run-details)
    * [run page](#run-page)
    * [details page](#details-page)
* [Project details](#project-details)
* [View](#view-summary)
* [Blacklist and Whitelist](#blacklist-and-whitelist)
* [Project Settings](#project-settings)
* [View Settings](#view-settings)
* [Limitations](#limitations)
* [Diff cli tool](#diff-cli-tool)
* [Future work](#future-work)

## Implementation details
The xml reports you recieve, should be post processed a bit, and compressed.  Compressed, as plugin is used with reports hundrets of megabytes large. And postprocessed as various engines generates various reports. Thus the xml files should be gathered into archives, which are later considered as suites:
![suites](https://user-images.githubusercontent.com/2904395/43016538-6c5141aa-8c53-11e8-8b6f-2eb45ebcaf01.png)
The level of granularity is up to you. The tar.gz archvies are later cached as two relatively small json files - one with listing for diff, one with stack traces of failures.  Latest cache is the properties file with total/run/passed/failed/error/skipp keys to be reused via https://github.com/judovana/jenkins-report-generic-chart-column (and used for quicker renderig of graphs of this plugin itself)

## Job run details

### run page
On a build page, simple ```JTREG Reports Total: X, Error: Y, Failed: Z``` message is printed, where **JTREG Reports** is link to  dretails page:

### details page
![details page](https://user-images.githubusercontent.com/2904395/43016541-6cb4cc5c-8c53-11e8-944b-cf1d274c492e.png)
Here yo can see several items:
* dummy **navigation** which allwos you to skip back and fwd. It do not check status of run, so it can return 404 on failed run
* **table**, with suites. Each suite have detailed  total/passed/failed/error skip details.  Table is followed by
* **listing of faiures**. The listing is clear, and contains failed tests only. Each failure contains expandable
    * **trace**. The listing is followd by
* **diff**. Diff is done first for added/removed suites. For unchanged suites, the diff of suite itself is done. You can immediately see exact **fixed** and **freshly failing** tests. After the diff, you can see 
* **full listing** of testsuite. The ful listing is stripped to 1000 lines due to performance. The output is generated by the rutines of [Diff cli tool](#diff-cli-tool), so please apologise a bit of cryptic "passed or missing" and "faile dor error" output. To see the full listing, you must use the [cli](#diff-cli-tool).
* on start of each section is cross roada allowing direct jumps to table, list of issues, diff and full listing
## Project details
![project details page](https://user-images.githubusercontent.com/2904395/43016540-6c95e4a4-8c53-11e8-8e59-db5b6b729b6b.png)
On the project page, you can see several graphs: 
* **number of failures**  shows how much test failed and how much had error. The graph is scalled, but in usual world, there is very small number of failures, and similarly small number of errors, so scale is not affected.
* **number of total tests**  shows how much test exists in suite and how much wasrun. The graph is scalled, but in usual world, there is very small number of skipped tests so both lines are of same value and so scale again should not be affected.
* **regression chart**  shows how much test had changed status. Green bar - number of tests which get fixed. Red bar, number of tests which started to fail. This graph is here to prevent overlook of X fixed tests and (same) X of freshly failing tests. IN such case, your total number of passes/failures is constant, and thus invisible on normla charts.

Each chart have detailed tooltip  and is capable of click which takes you directly to [details page](#details-page):
![tooltip](https://user-images.githubusercontent.com/2904395/43016542-6cd42700-8c53-11e8-9406-e0b3a908c60a.png)

## View summary
You can place jtreg charts also to jenkins view, so you can eyball all your testruns in one gaze.
![view summ](https://user-images.githubusercontent.com/2904395/43015875-21c739fc-8c51-11e8-9026-c84127628634.png)
Here the jtreg plugin is the left most graph. The two right most graphs are [charts from properties](https://github.com/judovana/jenkins-report-generic-chart-column)

## Blacklist and Whitelist
You could have noted, that the graphs are scaled. Sometimes it happens that fails 100x more tests then usually. This is killing the scale, and you can miss the regression. Such a build deserves  to be blacklisted once the issue is solved. On contrary, whitelist is here to allow you to comapre just some selected runs. Both lists are space separated list of regexes against job name (usually #NUMBER or some_custom_name). The lists are shared betwen project and view.

## Project Settings
![jtreg-project-settings](https://user-images.githubusercontent.com/2904395/43445509-dadbf3da-94a6-11e8-869b-44242a5a20fb.png)
Project settings are simple - you set glob expression for files to be considered as archives with results, and select how many tests you wish to show

## View Settings
![jtreg-view-settings](https://user-images.githubusercontent.com/2904395/43445508-dabd0682-94a6-11e8-824b-2609128dd016.png)
View settigns are fully inherited from project settings. So the only thing you do in view is to set the order - column for a chart

## Limitations
The imitations are clear from shared settings with all pros and cons and quite clumsy comparsion of exact jobs (w/b lists) and impossible comparsion between jobs.

## Diff cli tool
To workaround limitations, and add possibility to post-process results, the hpi and jars contains main class of ```hudson.plugins.report.jck.main.CompareBuilds``` whcih allows (based on the director with yor jobs) to comapre or list practically anything.  The launcher can look like:
```
set -x
set -e
# we need to get jenkins on classpath...
unpackDir=\$(dirname \$(mktemp))/$jenkinsWar
if [ ! -d \$unpackDir  ] ; then
  mkdir \$unpackDir
  unzip $jenkinsSrc -d \$unpackDir
fi
set +x
jars=\`find \$unpackDir | grep \\.jar \`
CP=""
for jar in \$jars ; do
  CP=\$CP\":\"\$jar
done
CP=\$CP\":\"$jenkins_main_dir/jenkins-report-jck-jar-with-dependencies.jar

set -x

/opt/jdk/bin/java -cp \$CP -Djenkins_home=$jenkins_main_home hudson.plugins.report.jck.main.CompareBuilds  $@
```
It can spwn plain tex, colored tex, or even html, so it i easy to be deployed as service (there is a wrapper for this too - ```hudson.plugins.report.jck.main.Service``` ) together with  jenkins.
```
options DIR1 DIR2 DIR3 ... DIRn
  or  
 options JOB_NAME-1 buildPointer1.1 buildPointer1.2  ... jobPointer1.N JOB_NAME-2 buildPointer2.1 buildPointer2.2  ... jobPointer2.N ... JOB_NAME-N ...jobPointerN.N
     If you use only jobname, then its builds will be listed:  
  options:  
 -output=color|html|html2
 default output is 'plain text'. 0-1 of -output is allowed.
 -view=all-tests|diff|diff-details|diff-list|diff-summary|diff-summary-suites|hide-misses|hide-negatives|hide-positives|hide-totals|info|info-hide-details|info-problems|info-summary|info-summary-suites
 default view is 'all'. 0-N of -view is allowed. Best view is  -view=diff-list  -view=info  -view=info-hide-details 
 job pointers are numbers. If zero or negative, then it is 0 for last one, -1 for one beofre last ...
 Unknow job will lead to listing of jobs.
 When using even number of build pointers, you can use -fill switch to consider them as rows
 Another strange argument is -keep-failed which will include failed/aborted/not-existing builds/dirs during listing.
```
The cli works with absolute and relative job IDs, and strictly cooperates with stdout/err (so consider  2>/dev/null sometimes)
* with no param it prints name
* with no job it prints jobs
* with no build it lists builds (of given job)
On everything else it do the listings and/or comparsion (or fails). Eg summary:
![cli1](https://user-images.githubusercontent.com/2904395/43448360-0b965e6e-94ae-11e8-986e-24faba2a8cce.png)

diff:
![cli-dif1-cli-dif3](https://user-images.githubusercontent.com/2904395/43448359-0b793672-94ae-11e8-9285-a837f4a4b8b9.png)

html view:
web-cli - ```hudson.plugins.report.jck.main.Service``` - is nothing more then wrapper around ```hudson.plugins.report.jck.main.CompareBuilds``` and is doing nothing more then resending stdout/err to browser request!! There is hardcoded port of 9090 in Sevice class.
![wb1-web3](https://user-images.githubusercontent.com/2904395/43450562-4346fab2-94b3-11e8-931a-bb26456d8aac.png)
Html output is much more clumsy, but the listing of switches and jobs is live, and also ajax is helping here a bit. Also yu can send results as URL, so it have its cases

## Future work
* To figure out how to make job-name based view on top of the shared settings.
* make cli more user friendly

This plugin depends on https://github.com/judovana/jenkins-chartjs-plugin
