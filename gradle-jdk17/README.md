# Snyk Gradle (jdk17)  Action

A [GitHub Action](https://github.com/features/actions) for using [Snyk](https://snyk.co/SnykGH) to check for
vulnerabilities in your Gradle-jdk17 projects. This Action is based on the [Snyk CLI][cli-gh] and you can use [all of its options and capabilities][cli-ref] with the `args`.

 > Note: The examples shared below reflect how Snyk github actions can be used. Snyk requires Python to have downloaded the dependencies before running or triggering the Snyk checks.
                          > The Python image checks and installs deps only if the manifest files are present in the current path (from where action is being triggered)
                          > 1. If pip is present on the current path , and Snyk finds a requirements.txt file, then Snyk runs pip install -r requirements.txt.
                          > 2. If pipenv is present on the current path, and Snyk finds a Pipfile without a Pipfile.lock, then Snyk runs pipenv update
                          > 3. If pyproject.toml is present in the current path and Snyk does not find poetry.lock then Snyk runs pip install poetry
                          >
                          > If manifest files are present under any location other root then they MUST be installed prior to running Snyk.

You can use the Action as follows:

```yaml
name: Example workflow for Python using Snyk
on: push
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/gradle-jdk17@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

## Properties

The Snyk Python Action has properties which are passed to the underlying image. These are passed to the action using `with`.

| Property | Default | Description                                                                                         |
| -------- | ------- | --------------------------------------------------------------------------------------------------- |
| args     |         | Override the default arguments to the Snyk image. See [Snyk CLI reference for all options][cli-ref] |
| command  | test    | Specify which command to run, for instance test or monitor                                          |
| json     | false   | In addition to the stdout, save the results as snyk.json                                            |

For example, you can choose to only report on high severity vulnerabilities.

```yaml
name: Example workflow for Python using Snyk
on: push
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/gradle-jdk17@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
```

## Uploading Snyk scan results to GitHub Code Scanning

Using `--sarif-file-output` [Snyk CLI flag][cli-ref] and the [official GitHub SARIF upload action](https://docs.github.com/en/code-security/secure-coding/uploading-a-sarif-file-to-github), you can upload Snyk scan results to the GitHub Code Scanning.

![Snyk results as a SARIF output uploaded to GitHub Code Scanning](../_templates/sarif-example.png)

The Snyk Action will fail when vulnerabilities are found. This would prevent the SARIF upload action from running, so we need to introduce a [continue-on-error](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstepscontinue-on-error) option like this:

```yaml
name: Example workflow for Python using Snyk
on: push
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/gradle-jdk17@master
        continue-on-error: true # To make sure that SARIF upload gets called
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --sarif-file-output=snyk.sarif
      - name: Upload result to GitHub Code Scanning
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: snyk.sarif
```

## Compatibility with `actions/set-up-java`

The official GitHub action to set up Java exports the environment variable `JAVA_HOME` which collides with the `JAVA_HOME` variable set inside the Snyk image.

This causes the Snyk step to fail as it is unable to resolve the `JAVA_HOME` directory inside the container.

In order to ignore the `JAVA_HOME` environment variable set by the `actions/set-up-java` we would need to enforce the value expected by the Snyk container as seen below:

```yaml
steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Set up Java
        uses: actions/setup-java@v3 # Exports JAVA_HOME variable incompatible with Snyk actions
        with:
          java-version: 17
          distribution: temurin
      - name: Set up Gradle
        uses: gradle/gradle-build-action@v2
        with:
          gradle-version: 7.3

      # Run your Gradle commands

      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/gradle-jdk17@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
          JAVA_HOME: /opt/java/openjdk # Enforce directory expected by the Snyk container
        with:
          args: --severity-threshold=high
```

Made with 💜 by Snyk

[cli-gh]: https://github.com/snyk/snyk 'Snyk CLI'
[cli-ref]: https://docs.snyk.io/snyk-cli/cli-reference 'Snyk CLI Reference documentation'
