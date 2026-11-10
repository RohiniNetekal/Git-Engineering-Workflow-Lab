# CI/CD for Java Backend APIs with GitHub Actions

> Independent learning material. This repository is a Git/workflow documentation lab, not a Spring Boot Maven application. Its active workflow validates Markdown. The Maven workflow below is a reusable template to copy into a real Java service repository; it is not presented as a build currently running here.

## 1. CI, continuous delivery and continuous deployment

- **Continuous Integration (CI):** every proposed change is built and checked automatically so integration problems are found before merge.
- **Continuous Delivery:** every accepted change can be released through a repeatable process, but production release may require a person or approval.
- **Continuous Deployment:** every change that passes the required controls is deployed automatically.

A backend pipeline usually looks like this:

~~~text
Feature branch
    |
Pull request
    |
Compile + unit tests + static checks
    |
Integration tests (database / Kafka / HTTP)
    |
Package JAR + build container image
    |
Publish immutable artifact
    |
Deploy to staging -> smoke checks
    |
Approval / policy gate
    |
Promote the same artifact to production
~~~

The important property is repeatability: the same source revision and build inputs should produce a traceable artifact. Do not rebuild different binaries for staging and production and assume they are identical.

## 2. Why use Maven's verify phase?

Maven's default lifecycle is sequential. Calling a later phase also executes the earlier phases:

- **validate:** check that the project model and required information are valid.
- **compile:** compile production source.
- **test:** run unit tests, normally through Maven Surefire.
- **package:** create the JAR/WAR.
- **verify:** run configured verification checks, including integration-test verification when Maven Failsafe is configured.

For a service repository, **./mvnw --batch-mode --no-transfer-progress clean verify** is a useful CI default. Prefer the checked-in Maven Wrapper so local development and CI use the repository's declared Maven distribution.

### Unit tests vs integration tests

- **Surefire** normally runs unit tests during the test phase. A test should isolate the unit under test and replace collaborators when that is the purpose of the test.
- **Failsafe** binds integration-test execution and verification to later lifecycle phases. Configure its integration-test and verify goals in the project POM.
- **Testcontainers** can start disposable dependencies such as a database or Kafka broker for integration tests. A test that merely mocks a repository is not a database integration test.

Important: running Maven verify does not magically create integration tests. Failsafe must be configured and the integration tests must follow the project's naming/configuration convention (for example, *IT). Do not mark a test as an integration test and then accidentally leave it outside the configured test includes.

Example Failsafe configuration fragment (manage the plugin version in the parent POM):

~~~xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>${maven-failsafe-plugin.version}</version>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
~~~

## 3. Reusable GitHub Actions workflow for a Maven service

Copy this template into a repository that contains pom.xml, the Maven Wrapper (mvnw and .mvn/wrapper), and tests. It tests a Java 17-compatible codebase on both Java 17 and 21. If the service uses Java 21-only APIs or language features, remove Java 17 from the matrix rather than expecting it to compile.

~~~yaml
name: Java CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: java-ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  verify:
    name: Verify on Java ${{ matrix.java }}
    runs-on: ubuntu-latest
    timeout-minutes: 25
    strategy:
      fail-fast: false
      matrix:
        java: ['17', '21']

    steps:
      - name: Check out source
        uses: actions/checkout@v7

      - name: Set up Eclipse Temurin
        uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: ${{ matrix.java }}
          cache: maven

      - name: Verify project
        run: ./mvnw --batch-mode --no-transfer-progress clean verify
~~~

What each part does:

1. pull_request validates proposed changes before merge; push validates the protected branch after merge.
2. The Java matrix catches accidental use of APIs newer than the declared baseline. A matrix increases cost, so test only the versions the project actually supports.
3. cache: maven reuses downloaded dependencies. The cache is an optimization, not a source of truth; the build must still resolve dependencies correctly.
4. clean verify runs the configured build and test lifecycle. Do not replace it with package -DskipTests in the quality gate.
5. permissions: contents: read avoids giving a test job write access it does not need.
6. The concurrency group cancels obsolete runs for the same branch/ref when newer commits arrive.

If the project has no Maven Wrapper, install/use a known Maven version and run Maven with --batch-mode --no-transfer-no-progress clean verify. Adding and committing the wrapper is generally more reproducible.

## 4. Add integration tests without hard-coding CI infrastructure

A Spring Boot integration test can use Testcontainers to launch a disposable database or Kafka broker. On GitHub-hosted Linux runners, Docker is available to the job, so the test can own the lifecycle of its containers.

A useful test suite for a payment/merchant-style API might verify:

- two concurrent requests with the same idempotency key create one business operation;
- a different request body with the same key returns a conflict;
- a database unique constraint, not only an in-memory check, prevents duplicate rows;
- an outbox row commits with the business change;
- a consumer safely handles the same event twice;
- invalid JWT signatures, wrong issuer and wrong audience are rejected.

Use real dependencies only where their behavior matters. Keep ordinary service unit tests fast with Mockito; use Testcontainers for persistence, broker and transaction behavior that mocks cannot prove.

## 5. Build and promote a container image

After verify passes, a container build can validate the Dockerfile:

~~~bash
docker build --tag merchant-api:${GITHUB_SHA} .
~~~

For a real release pipeline:

- build once from a trusted commit;
- tag the image with the commit SHA and record its digest;
- scan the image and dependencies;
- publish only from a trusted branch/tag workflow;
- deploy the same digest to staging and production;
- run health/readiness and business smoke checks after deployment.

Do not publish an image from an untrusted fork pull request. A pull-request validation job should not need registry credentials.

## 6. Separate deployment permissions from build permissions

Build and test jobs should normally be read-only. A deployment job is different: it may need permission to request a short-lived cloud identity token and access a protected environment.

For cloud deployment, prefer the cloud provider's OIDC federation with GitHub Actions over a long-lived access key stored as a repository secret. Give id-token: write only to the job that needs it, configure an environment such as staging or production, and use environment approvals/protection rules for production.

Keep provider-specific login and deployment steps in a separate workflow because AWS, Azure and Google Cloud use different federation actions and trust policies. Never print tokens or secrets in logs.

## 7. Workflow security checklist

- Use pull_request for normal build/test validation.
- Do not use pull_request_target to check out and execute untrusted pull-request code.
- Set explicit least-privilege permissions; start with read-only.
- Keep deployment credentials out of build/test jobs.
- Use protected environments and short-lived OIDC credentials for cloud deployment.
- Pin third-party actions to reviewed full commit SHAs in hardened production repositories; update pins through reviewed dependency updates.
- Do not interpolate untrusted pull-request titles, branch names or body text directly into shell commands.
- Do not upload secrets, environment files or production data as build artifacts.

The active documentation workflow in this repository follows the same principle: it needs only contents: read and no secrets.

## 8. Common mistakes and trade-offs

- Running only Maven package: configured integration verification may not run. Use Maven verify.
- Using -DskipTests in CI: a green build may contain untested changes. Run the required test phases.
- Mocking every database call: transaction, SQL and constraint behavior remain untested. Add focused Testcontainers integration tests.
- Building a different image per environment: staging and production may run different bits. Promote the same image digest.
- Giving every job write permissions: this increases the impact of compromised dependencies or workflow code. Declare least-privilege permissions.
- Putting cloud keys in repository secrets: long-lived credentials can be exposed or copied. Prefer OIDC and short-lived credentials.
- Retrying every failed test: this hides race conditions and nondeterministic tests. Fix the cause; retry only known transient infrastructure operations.

## 9. Troubleshooting a failed pipeline

1. **Compilation failure:** compare the local JDK, CI matrix version and Maven compiler release.
2. **Dependency resolution failure:** inspect repository credentials and dependency cache keys; retry only after checking whether the repository is temporarily unavailable.
3. **Unit-test failure:** read the first failing assertion and confirm test isolation and deterministic data.
4. **Testcontainers failure:** check Docker availability, container startup logs, image architecture and readiness wait strategy.
5. **Integration test not executed:** verify Failsafe goals and test naming/includes.
6. **Docker build failure:** compare the build context, .dockerignore, multi-stage paths and runtime base image.
7. **Deployment failure:** inspect environment approval, OIDC trust conditions, role permissions and health checks without printing credentials.

## 10. Interview questions

1. What is the difference between CI, continuous delivery and continuous deployment?
2. Why is Maven verify often preferred over Maven package in CI?
3. What is the difference between Maven Surefire and Failsafe?
4. Why can Mockito tests not replace every database integration test?
5. How does Testcontainers help reproduce database or Kafka behavior?
6. Why should a build job use a read-only GITHUB_TOKEN?
7. What is the risk of checking out and executing fork code in pull_request_target?
8. Why should staging and production receive the same container digest?
9. How does OIDC reduce the risk of long-lived cloud credentials?
10. How would you investigate a test that passes locally but fails intermittently in CI?

## 11. Practical exercise

1. In a separate Spring Boot Maven project, add a Maven Wrapper and one deterministic JUnit test.
2. Add the Java CI workflow and confirm a pull request runs clean verify.
3. Configure Failsafe and add one Testcontainers integration test for a real persistence constraint.
4. Add a Dockerfile and make the image build depend on successful tests.
5. Publish an image only from a trusted main/tag workflow.
6. Add a staging environment and OIDC-based deployment; require approval before production.
7. Deliberately break a unit test and an integration test, then explain how the workflow reports each failure.

## Official references

- [GitHub Actions workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [GitHub Actions security: pull_request_target](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target)
- [GitHub Actions: building and testing Java with Maven](https://docs.github.com/en/actions/tutorials/build-and-test-code/java-with-maven)
- [actions/setup-java](https://github.com/actions/setup-java)
- [Apache Maven build lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle)
