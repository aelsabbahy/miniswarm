FROM instavote/vote
MAINTAINER Ahmed Elsabbahy <elsabbahyahmed@yahoo.com>

# Install goss
RUN apk add --no-cache --virtual=goss-dependencies curl ca-certificates && \
    curl -fsSL https://goss.rocks/install | sh && \
    apk del goss-dependencies

# Add our tests
COPY goss/ /goss/

# mm.. healthchecks
HEALTHCHECK --retries=20 --interval=5s --timeout=2s CMD goss -g /goss/goss.yaml validate
