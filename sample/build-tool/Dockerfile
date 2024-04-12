FROM mgoltzsche/podman:4.9.4-minimal
COPY ./kustomize /usr/local/bin/kustomize
RUN apk update \
  && apk upgrade \
  && apk add --no-cache --update python3 py3-pip coreutils bash \
  && rm -rf /var/cache/apk/* \
  && pip3 install pyyaml==5.3.1 \
  && pip3 install -U awscli \
  && apk --purge -v del py3-pip \
  && apk add git
