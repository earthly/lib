VERSION 0.6

INSTALL_DIND:
    COMMAND
    COPY +install-dind-script/install-dind.sh /tmp/install-dind.sh
    RUN /tmp/install-dind.sh

install-dind-script:
    FROM alpine:3.17
    COPY ./install-dind.sh ./
    SAVE ARTIFACT ./install-dind.sh

test:
    BUILD +test-install-dind-amd64
    BUILD ./ssh+test-add-known-hosts

test-install-dind-amd64:
    BUILD --platform=linux/amd64 +test-install-dind-for-image \
        --base_image=docker:dind \
        --base_image=alpine:latest \
        --base_image=debian:stable \
        --base_image=debian:stable-slim \
        --base_image=ubuntu:latest \
        --base_image=amazonlinux:1 \
        --base_image=amazonlinux:2 \
        --base_image=earthly/dind:alpine \
        --base_image=earthly/dind:ubuntu

test-install-dind-arm64:
    BUILD --platform=linux/arm64 +test-install-dind-for-image \
        --base_image=docker:dind \
        --base_image=alpine:latest \
        --base_image=ubuntu:latest \
        --base_image=earthly/dind:alpine \
        --base_image=earthly/dind:ubuntu

test-install-dind-for-image:
    ARG base_image
    FROM "$base_image"

    # docker-compose will fail with:
    #     Error response from daemon: Invalid container name (-hello-1), only [a-zA-Z0-9][a-zA-Z0-9_.-] are allowed
    # if docker-compose.yml is in the root directory, due to the --project flag defaulting to "".
    # This bug does not exist in the official earthly/dind:alpine image.
    WORKDIR /dind-test

    DO +INSTALL_DIND
    RUN echo "
version: \"3\"
services:
    hello:
        image: hello-world:latest
    " >./docker-compose.yml
    WITH DOCKER --compose docker-compose.yml
        RUN true
    END
