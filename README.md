# q-app

This is a small quarkus demo project that can be used to deploy to openshift

## Build and run in the same container

To deploy this application as a native build using the Quarkus S2I image do the following

    oc new-project demo-s2i

    oc new-app --name=q-app quay.io/quarkus/centos-quarkus-native-s2i:graalvm-1.0.0-rc15~https://github.com/tqvarnst/q-app.git

    oc cancel-build bc/q-app

    oc patch bc/q-app -p '{"spec":{"resources":{"limits":{"cpu":"4", "memory":"4Gi"}}}}'
    
    oc start-build q-app

    # wait for the build to finish (web console or `oc logs -f bc/q-app`)

    oc expose svc q-app

The reason that we are cancelling the build shortly after creating it (in the `oc new-app` command) is because the default resource that OpenShift uses to build a native application using GraalVM isn't enough. Therefor we first patch the build config giving it more resources and then start the build again.

## Deploy as minimal runtime container

In the Deploy as S2I example the container that builds the native application is also running the application. This means that the image contains both OpenJDK, GraalVM, Maven, etc even though that is not required to run the application. 

An alternative way of deploying this is to use a chained build process where we have one build process to build the native application and another one that creates a minimal container that only consists of the base container and the native application it self. 

The following commands will create a chained build.

    oc new-project demo-runtime

    oc new-build --name=q-app-build quay.io/quarkus/centos-quarkus-native-s2i:graalvm-1.0.0-rc15~https://github.com/tqvarnst/q-app.git 

    oc cancel-build bc/q-app-build

    oc patch bc/q-app-build -p '{"spec":{"resources":{"limits":{"cpu":"4", "memory":"4Gi"}}}}'

    oc start-build q-app-build

    oc new-build --name=q-app \
        --docker-image=registry.access.redhat.com/ubi7-dev-preview/ubi-minimal \
        --source-image=q-app-build \
        --source-image-path='/home/quarkus/application:.' \
        --dockerfile=$'FROM registry.access.redhat.com/ubi7-dev-preview/ubi-minimal:latest\nCOPY application /application\nCMD /application\nEXPOSE 8080' \
        --allow-missing-imagestream-tags

    # wait for the build to finish (web console or `oc logs -f bc/q-app`)

    oc new-app q-app

    oc expose svc q-app