ARG BALENA_ARCH

FROM balenalib/${BALENA_ARCH}-debian-node:12-build as build

WORKDIR /usr/src/app

COPY . .

# dev dependencies are required for build:fast
# --unsafe-perm is not needed because of global /usr/local/etc/npmrc
RUN npm install

RUN npm run build:fast

# remove dev dependencies after build:fast
RUN npm prune --production

FROM balenalib/${BALENA_ARCH}-debian-node:12-run

WORKDIR /usr/src/app

COPY --from=build /usr/src/app/ .

ENV PATH $PATH:/usr/src/app/bin

# fail early if balena binary won't run
RUN balena --version

# https://github.com/balena-io/balena-cli/blob/master/INSTALL-LINUX.md#additional-dependencies
RUN install_packages avahi-daemon ca-certificates docker.io openssh-client

COPY docker/docker-init.sh init.sh

RUN chmod +x ./init.sh

ENTRYPOINT [ "./init.sh" ]

CMD [ "--help" ]
