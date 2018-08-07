# Example for the K.I.S.S.

Java, wrapped in Docker, running in k8s on GCP.

## Notes

### Create new repository in GitHub

### Generate Gradle Spring Boot jar project with web
Run local Gradle build to confirm the project is running


### Add Travis
1. Add .travis.yml for Java app 
    ```
    language: java
    ```
1. Enable the repo in Travis config
1. Push the project to git and watch the Travis build.
    ```
    git init
    git add.
    git commit -m "Vanilla project"
    git remote add origin github_url
    git remote -v
    git push -u origin master
    ```
    
### Add Artifactory
1. Install jfrog-cli
    ```
    brew install jfrog-cli-go
    ```
2. Configure jfrog-cli to work with Artifactory
    ```
    jfrog rt config kiss --url=https://kiss.jfrog.io/kiss
    ```
3. Generate gradle yaml configuration and add it to git
    ```
    jfrog rt gradlec gradleSpec.yml
    ```
4. Install travis-cli
    ```
    gem install travis
    ```
5. Encrypt and add the encrypted password to travis.yml 
    ```
    travis encrypt ARTIFACTORY_KEY=password --add
    ```   
1. Add caching to prevent re-download of the wrapper and the dependencies
    ```
    cache:
        directories:
        - $HOME/.gradle/caches
        - $HOME/.gradle/wrapper
    ```
6. Configure jfrog-cli in Travis in `before_install`
    ```
    before_install:
    - wget https://dl.bintray.com/jfrog/jfrog-cli-go/1.17.1/jfrog-cli-linux-amd64/jfrog
    - chmod +x jfrog
    - ./jfrog rt config kiss --url=https://kiss.jfrog.io/kiss --user=admin --password=$ARTIFACTORY_KEY --interactive=false
    ```
1. Cancel the install phase
    ```
    install: true
    ```
7. Add gradle build to travis.yml `script` section
    ```
    script:
    - ./jfrog rt gradle "build artifactoryPublish" gradleSpec.yml --build-name=$TRAVIS_REPO_SLUG --build-number=$TRAVIS_BUILD_NUMBER
    ```
8. Add environment and git information to the build-info and deploy the build info
    ```
    - ./jfrog rt build-collect-env $TRAVIS_REPO_SLUG $TRAVIS_BUILD_NUMBER
    - ./jfrog rt build-add-git $TRAVIS_REPO_SLUG $TRAVIS_BUILD_NUMBER
    - ./jfrog rt build-publish --build-url="https://travis-ci.org/$TRAVIS_REPO_SLUG/builds/$TRAVIS_BUILD_ID" $TRAVIS_REPO_SLUG $TRAVIS_BUILD_NUMBER
    ```
11. Remove the `repositories` closure from the `build.gradle`
10. Push the changes (`travis.yml`, `gradleSpec.yml`, `build.gradle`) and watch the Travis build.

### Configure Docker registry in Artifactory
1. Add set of Docker repos
2. Add https://gcr.io as a gcr-docker-remote repo with token authentication disabled, and add it to the virtual

### Add Dockerfile
1. Create a simple `Dockerfile`. It uses the distroless base image from Google and adds the Gradle output as the applicative dependency.
    ```
    FROM kiss-docker.jfrog.io/distroless/java

    COPY build/libs/*.jar /app.jar
    ENTRYPOINT ["java", "-jar", "/app.jar"]
    ```
1. Add `.dockerignore` that will ignore everything in the working dir, except the needed java file
    ```
    **
    !build/libs/*

    ```
1. Login to Artifactory Docker registry in `before_install`
    ```
    - docker login --username admin --password $ARTIFACTORY_KEY kiss-docker.jfrog.io
    ```
1. Build the Docker image
    ```
    - docker build -t kiss-docker.jfrog.io/$TRAVIS_REPO_SLUG:$TRAVIS_BUILD_NUMBER .
    ```
    
1. Add the Java file as a dependency to the build info
    ```
    - ./jfrog rt build-add-dependencies $TRAVIS_REPO_SLUG $TRAVIS_BUILD_NUMBER "build/libs/*.jar"
    ```
1. Push the docker image to Artifactory
    ```
    - ./jfrog rt docker-push kiss-docker.jfrog.io/$TRAVIS_REPO_SLUG:$TRAVIS_BUILD_NUMBER docker --build-name=$TRAVIS_REPO_SLUG --build-number=$TRAVIS_BUILD_NUMBER
    ```
1. 10. Push the changes (`Dockerfile`, `.dockerignore`, `.travis.yml`) and watch the Travis build.

### Add Kubernetes
1. Create `kubernetes/` directory: `mkdir kubernetes`
1. Generate a Deployment YAML `kubectl run sample-app --image=kiss-docker.jfrog.io/kiss-the-frog/test-two:latest --dry-run -oyaml > sample-app-deployment.yaml`
1. Add a new Docker Registry secret `kubectl create secret docker-registry artifactory-registry --docker-username=admin --docker-password=... --docker-email=me@not.val.id --docker-server=kiss-docker.jfrog.io`
1. Add `ImagePullSecrets` to reference to Docker Registry secret.
    ```yaml
    apiVersion: extensions/v1beta1
    kind: Deployment
    ...
    spec:
      ...
      template:
      ...
        spec:
          imagePullSecrets:
          - name: artifactory-registry
          containers:
            ...
    ```
1. Generate the Service `kubectl expose -f sample-app-deployment.yaml --type=LoadBalancer --port=8080 --target-port=8080 --dry-run -oyaml > sample-app-svc.yaml`
1. Deploy the service: `kubectl apply -f sample-app-deployment.yaml`
1. Create a GCP Service Account, with limited role/permission (e.g., Kubernetes Developer)
1. Generate a JSON key for the service account
1. Encrypt the JSON key with Travis: `travis encrypt-file travis-service-account.json --add`
1. Update Travis to download `gcloud`
    ```yaml
    cache:
      directories:
      - $HOME/google-cloud-sdk

    before_install:
    - export CLOUDSDK_CORE_DISABLE_PROMPTS=1
    - if [ ! -d "$HOME/google-cloud-sdk/bin" ]; then
      rm -rf $HOME/google-cloud-sdk;
      curl https://sdk.cloud.google.com | bash;
    fi
    - source $HOME/google-cloud-sdk/path.bash.inc
    ```
1. Set the default project, and activate the service account.
    ```yaml
    before_install:
    ...    
    - gcloud config set core/project kiss-jfrog
    - gcloud auth activate-service-account --key-file=$TRAVIS_BUILD_DIR/travis-ci-service-account.json
    ```
1. Install `kubectl` command line, and fetch the GKE credential
    ```yaml
    before_install:
    ...
    - gcloud components install kubectl
    - gcloud container clusters get-credentials kiss-jfrog-cluster --zone=us-central1-a
    ```
1. Update image to the image from the build and push to Kubernetes
    ```yaml
    script:
    ...
    - kubectl set image -f kubernetes/sample-app-deployment.yaml sample-app=kiss-docker.jfrog.io/$TRAVIS_REPO_SLUG:$TRAVIS_BUILD_NUMBER
    ```
