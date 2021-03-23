
INSTALL_DIND:
    COMMAND
    ARG script_target=github.com/earthly/lib:main+install-dind-script
    COPY "${script_target}/install-dind.sh" /tmp/install-dind.sh
    RUN /tmp/install-dind.sh

install-dind-script:
    FROM alpine:3.13
    COPY ./install-dind.sh ./
    SAVE ARTIFACT ./install-dind.sh

test:
    BUILD +test-install-dind

test-install-dind:
    BUILD +test-install-dind-for-image \
        --base_image=docker:dind \
        --base_image=alpine:latest \
        --base_image=debian:stable \
        --base_image=debian:stable-slim \
        --base_image=ubuntu:latest \
        --base_image=amazonlinux:1 \
        --base_image=amazonlinux:2 \
        --base_image=earthly/dind:alpine \
        --base_image=earthly/dind:ubuntu

test-install-dind-for-image:
    ARG base_image
    FROM "$base_image"
    DO +INSTALL_DIND --script_target=./
    RUN echo "
version: \"3\"
services:
    hello:
        image: hello-world:latest
    " >./docker-compose.yml
    WITH DOCKER --compose docker-compose.yml
        RUN true
    END
