## Getting started with the CodeQL CLI

When you want to generate a CodeQL database locally and run the pre-compiled queries against it, this is the way to go.

First let's download the CodeQL bundle!

### Pre-requisites

You will need to make sure you have the GitHub CLI installed. For more information on how to install the CLI, check out this installation [doc](https://github.com/cli/cli#installation).

### Install the extension

Using the GitHub CLI, we will install the codeql cli,

  1. `gh extensions install github/gh-codeql`
  2. `gh codeql set-version latest`

Check to make sure you can use the CodeQL CLI

```
gh codeql --version
```

## Using the CodeQL CLI

Now we need to use the CodeQL CLI on an actual repository. Let's start here with our [GHAS training material](https://github.com/Coveros/ghas-certification-bootcamp).
There are multiple languages being used here, so for the purposes of this tutorial let's try to scan the JavaScript portions of the codebase. 

Clone this repository and `cd` into it.

### Install the JavaScript Bundle

We will need to download the latest JavaScript queries to scan the code with. In your terminal, run `gh codeql pack download codeql/javascript-queries`

### codeql database create

The first thing we need to do when it comes to CodeQL analysis is to create a CodeQL database. 
When it comes to interpreted languages and Go, CodeQL will use an autobuild.sh script that will extract the source code and create a snapshot database. 
When it comes to compiled languages, CodeQL needs to build the source code in order to trace the build and create a snapshot database of it. 
You can rely on the autobuild.sh script as well, or you can supply your own build instructions via the `--command` flag, which can be used when invoking the `codeql database create` command.
Please review this [list](https://codeql.github.com/docs/codeql-overview/supported-languages-and-frameworks/) of currently supported languages and frameworks.


```bash
gh codeql database create db --language=javascript
```

CodeQL will create the `db` directory and will choose the autobuild.sh script for the specified languages in order to begin the extraction process.
CodeQL will also finalize the database at the specified `db` directory. Within your codeql database directory (in this case `db`) 
you should notice a db-javascript directory which contains the db schemes and src.zip file with the source that was extracted.

#### Optional: Importing the CodeQL Database into Visual Studio Code
You can actually take this database and import it into your Visual Studio Code workspace.
To get started on that, please go to this [repository](https://github.com/github/vscode-codeql-starter) and follow the instructions on how to set up the CodeQL starter workspace and install the CodeQL plugin.
Once you have the CodeQL plugin installed, import the database you created in this step and try to run a JavaScript query against the database.


### codeql database analyze

Now that we have a database to work with, let's run some queries against it! GitHub offers three types of CodeQL query suites:

- `$CODEQL_SUPPORT_LANGUAGE-code-scanning.qls`
- `$CODEQL_SUPPORT_LANGUAGE-security-extended.qls`
- `$CODEQL_SUPPORT_LANGUAGE-security-and-quality.qls`

By default when we scan with `codeql/javascript-queries` it will default to `javascript-code-scanning.qls`.

```bash
gh codeql database analyze db --format=sarif-latest --output=codeql-javascript-results.sarif codeql/javascript-queries
```

You will see the queries being evaluated. When this process is done, a SARIF should have been created. The SARIF contains results from the analysis. 
If the results array is empty, it means no results were found. If you want to view the SARIF, you can use `jq` to parse through it, or you can use a SARIF Viewer, such as this [one](https://marketplace.visualstudio.com/items?itemName=WDGIS.MicrosoftSarifViewer). Also if you have the `vs-codeql-starter` [workspace](https://github.com/github/vscode-codeql-starter), you can run particular queries against an imported CodeQL database and see the analysis in the IDE.

To scan your code using the other query suites, you just need to append that to the original command

```bash
codeql database analyze db --format=sarif-latest --output=codeql-javascript-results.sarif codeql/javascript-queries:codeql-suites/javascript-security-extended.qls
```

#### Things to note
- When dealing with multiple analyses for the same commit (whether you're analysing multiple languages or have parallelized builds for a monorepo), make sure to use the `--sarif-category` flag to categorize the analyses.
Failure to do so, in particular on a pull request, can cause confusion in that Code Scanning may not be able to detect a baseline analysis to compare the PR results.
- Use this [endpoint](https://docs.github.com/en/rest/reference/code-scanning#list-code-scanning-analyses-for-a-repository) to list the CodeQL analyses of a repository, so that you can inspect the category for each analysis.
  - This is especially important for the next step.

### codeql github upload-results 

This step is typically used when you want to see the SARIF in the Code Scanning alerts UI. It's typically used when you want to post results to the default branch of a repository for the first time (baseline analysis) or to a pull request to see any security alert annotations.

Here are some advanced things to note:
- When posting the analysis for the first time to a default analysis, make sure you define a `--sarif-category`. That way the analyses for subsequent pull requests can also share the same category value. 
Note that this kind of depends on how you're running the builds (whether or not you've broken down a monorepo into separate analyses or you have multiple scans due to multiple languages) but typically just starting out, 
just make sure to have the same category value for subsequent scans, so that Code Scannning can easily figure out what the basline analysis is to compare subsequent analyses.

The `--ref` and `--commit` flag combinations can be one of the following:
- `refs/pulls/<pull request number>/merge` + MERGE commit
- `refs/heads/<branch name>` + HEAD commit
  - ` curl -H "Accept: application/vnd.github.v3+json" \\n  -H "Authorization: token $GH_TOKEN" \\n  https://api.github.com/repos/<org-name>/<repo-name>/pulls/<pull-request-number> | jq '.merge_commit_sha'`
  - The merge commit is a commit created to make sure PR checks are run; this commit doesn't exist in the actual source tree/`git log`.

If you are supplying the `--commit` flag, make sure you use the full commit hash and not the shortened one

```bash
gh codeql github upload-results --repository=$GITHUB_REPOSITORY --ref=$GITHUB_REF --commit=$GITHUB_SHA --sarif=codeql-javascript-results.sarif --github-auth-stdin=<YOUR TOKEN>
```

After running this command, you should see results on the Code Scanning alerts page for the particular `ref`. If you posted the analysis for a pull request, if there are any alerts, you should see a `Code scanning results` check too.

### References:
- https://codeql.github.com/docs/codeql-cli/manual/database-create/
- https://codeql.github.com/docs/codeql-cli/manual/database-analyze/
- https://codeql.github.com/docs/codeql-cli/manual/github-upload-results/
- https://docs.github.com/en/code-security/code-scanning/using-codeql-code-scanning-with-your-existing-ci-system/configuring-codeql-cli-in-your-ci-system#about-generating-code-scanning-results-with-codeql-cli
