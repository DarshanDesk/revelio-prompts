GitHub Copilot: Prompt Cookbook for Jenkins-to-Tekton Migration

0. Setup & Workspace

"Hello, please act as my expert pair programmer for a Jenkins-to-Tekton migration, following all the rules in my custom instructions."

"Let's start by setting up the shared state. Generate a Tekton PersistentVolumeClaim YAML manifest for a Workspace. Name it ci-source-pvc and give it 1Gi of storage with ReadWriteOnce access."

1. Converting Jenkins Stages to Tekton Tasks

Prompt 1.1: Simple sh block conversion

"I have this stage from my Jenkinsfile. Convert it to a complete Tekton Task YAML.

The Task should be named build-python-algorithm.

It must accept one param named algorithm-path.

It must use one workspace named source.

Use the python:3.9-slim image for the step.

All sh commands should be in one script block.

stage('Build Algorithm') {
    steps {
        echo "Building algorithm in ${params.ALGO_PATH}"
        sh 'cd ${params.ALGO_PATH}'
        sh 'python setup.py sdist'
        sh 'mkdir -p ../artifacts'
        sh 'mv dist/*.tar.gz ../artifacts/'
        sh 'echo "Artifact moved to ../artifacts"'
    }
}
```"

### Prompt 1.2: Multi-step `sh` block conversion

"Here is a more complex Jenkins stage. Convert this to a Tekton `Task` named `test-and-build`.
- This `Task` needs two `Steps`.
- The first `Step` (named `test`) should run the `pytest` command. Use the `python:3.9-slim` image.
- The second `Step` (named `build`) should run the `tar` commands. Use the `alpine:latest` image.
- Both steps must use the `algorithm-path` `param` and the `source` `workspace`.
- Ensure the `build` step correctly `cd`s to the workspace and algorithm path.

```groovy
stage('Test and Package') {
    steps {
        sh 'cd ${params.ALGO_PATH} && pip install -r requirements.txt && pytest'
        sh 'cd ${params.ALGO_PATH}'
        sh 'tar -czf ${params.ALGO_NAME}.tar.gz . --exclude=*.pyc --exclude=__pycache__'
        sh 'mv ${params.ALGO_NAME}.tar.gz ../../artifacts'
    }
}
```"

### Prompt 1.3: Analyzing a Jenkins `sh` script

"@workspace /explain this Jenkins `sh` script. What dependencies does it have? What container image would be best for a Tekton `Step` that needs to run this?

```sh
#!/bin/bash
set -e
echo "Logging in to GCR..."
gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
echo "Building and pushing image..."
gcloud builds submit --tag gcr.io/my-project/$ALGO_NAME:$VERSION .
echo "Build complete."
```"

## 2. Creating the `Pipeline`

### Prompt 2.1: Initial `Pipeline` Scaffolding

"Create a Tekton `Pipeline` named `python-algo-pipeline`.
- It must accept two `params`: `git-url` and `algorithm-path`.
- It must define one `workspace`: `source-repo`."

### Prompt 2.2: Adding Tasks to the `Pipeline`

"I have two `Tasks` ready: `fetch-repository` (from Tekton Hub) and `build-python-algorithm` (from Prompt 1.1).

Update the `python-algo-pipeline` to:
1.  Use the `tektoncd/pipeline` Hub `Task` for `git-clone` to fetch the repo. Name this pipeline task `clone`.
2.  Add our `build-python-algorithm` `Task`. Name this pipeline task `build`.
3.  Ensure the `build` task runs *after* the `clone` task using `runAfter`.
4.  Wire up the `Pipeline` `params` and `workspaces` to both `Tasks` correctly.
    - `clone` needs the `git-url` param and `source-repo` workspace.
    - `build` needs the `algorithm-path` param and `source-repo` workspace."

## 3. Running the `Pipeline`

### Prompt 3.1: Creating a `PipelineRun`

"Generate a `PipelineRun` YAML named `run-lona-g1`.
- It should execute the `python-algo-pipeline`.
- Set the `git-url` param to `https://github.com/my-org/my-algorithms.git`.
- Set the `algorithm-path` param to `LONA/LONA_G1`.
- Bind the `source-repo` workspace to our PVC named `ci-source-pvc`.
- Use `serviceAccountName: tekton-bot`"

### Prompt 3.2: Troubleshooting a `PipelineRun`

"My `PipelineRun` failed. Here is the YAML for the `PipelineRun` and the `TaskRun` that failed. Can you analyze the logs and tell me the likely cause?

(Paste `PipelineRun` YAML)
(Paste `TaskRun` YAML, especially the `status.steps` block with logs)"
