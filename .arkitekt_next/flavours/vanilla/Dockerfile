# Copyright 2024 The MathWorks, Inc.

# This Dockerfile allows you to build either the MATLAB Container image or the MATLAB Deep Learning Container image.

# To specify which MATLAB release to install in the container, edit the value of the MATLAB_RELEASE argument.
ARG MATLAB_RELEASE="R2021b"

# Specify the additional list of toolboxes and support packages to install into the
# mathworks/matlab-deep-learning Docker image.
ARG ADDITIONAL_PRODUCTS=""

# Specify any additional flags used to customise the `mpm` installation process.
ARG MPM_ADDITIONAL_FLAGS=""

# Specify the base repository to pull MATLAB dependencies from.
ARG MATLAB_DEPS_IMAGE=""

# Specify the URL of the managed version of MathWorks Service Host.
ARG MATHWORKS_SERVICE_HOST_INSTALL_URL="https://www.mathworks.com/MathWorksServiceHost/glnxa64/install_managed_msh.sh"

# Specify any additional packages to install via apt-get.
ARG APT_ADDITIONAL_PACKAGES="git python3.8-venv libpython3.8 build-essential"
ARG APT_ADDITIONAL_PACKAGES_VNC=""

# Specify MATLAB Install Location.
ARG MATLAB_INSTALL_LOCATION="/opt/matlab/${MATLAB_RELEASE}"

# When you start the build stage, this Dockerfile by default uses the Ubuntu-based matlab-deps image.
# To check the available matlab-deps images, see: https://hub.docker.com/r/mathworks/matlab-deps

#FROM ${MATLAB_DEPS_IMAGE}:${MATLAB_RELEASE} AS build
FROM ubuntu:20.04 AS build

LABEL maintainer="The MathWorks, Inc."

COPY base-dependencies.txt /tmp/base-dependencies.txt

RUN export DEBIAN_FRONTEND=noninteractive \
    && apt-get update \
    && apt-get install --no-install-recommends -y `cat /tmp/base-dependencies.txt` \
    && apt-get clean && apt-get -y autoremove \
    && rm -rf /var/lib/apt/lists/*

RUN [ -d /usr/share/X11/xkb ] || mkdir -p /usr/share/X11/xkb

# Install patched glibc - See https://github.com/mathworks/build-glibc-bz-19329-patch
# Note: base-dependencies.txt includes libcrypt-dev and linux-libc-dev to enable installation of patched -dev packages
WORKDIR /packages
RUN export DEBIAN_FRONTEND=noninteractive &&\
    wget -q https://github.com/mathworks/build-glibc-bz-19329-patch/releases/download/ubuntu-focal/all-packages.tar.gz &&\
    tar -x -f all-packages.tar.gz \
        --exclude glibc-*.deb \
        --exclude libc6-dbg*.deb &&\
    apt-get install --yes --no-install-recommends ./*.deb &&\
    rm -fr /packages
WORKDIR /

ARG MATLAB_RELEASE
ARG MATLAB_INSTALL_LOCATION
ARG MPM_ADDITIONAL_FLAGS
ARG MATHWORKS_SERVICE_HOST_INSTALL_URL
ARG APT_ADDITIONAL_PACKAGES
ARG APT_ADDITIONAL_PACKAGES_VNC

# Install extra tools not included in `matlab-deps`.
RUN export DEBIAN_FRONTEND=noninteractive \
    && apt-get -qq update \
    && apt-get -qq install --yes --no-install-recommends \
    apt-utils \
    # Requirements for mpm
    ca-certificates \
    # Requirements for VNC
    libglu1-mesa \
    libosmesa6 \
    xfce4 \
    xscreensaver \
    # Requirements for matlab-proxy and noVNC
    python3 \
    python3-pip \
    pipx \
    xvfb \
    # Utility Software
    tini \
    sudo \
    curl \
    wget \
    nano \
    xfce4-terminal \
    psmisc \
    less \
    ${APT_ADDITIONAL_PACKAGES} \
    && apt-get remove -y \
    tumbler \
    pulseaudio \
    gvfs \
    gnome-screensaver \
    && apt-get clean \
    && apt-get -y autoremove \
    && rm -rf /var/lib/apt/lists/*

################################################################################
# Setup VNC
################################################################################

RUN export DEBIAN_FRONTEND=noninteractive \
    && apt-get -qq update \
    && apt-get -qq install --yes --no-install-recommends \
    tigervnc-standalone-server \
    tigervnc-common \
    dbus-x11 \
    ${APT_ADDITIONAL_PACKAGES_VNC} \
    && apt-get clean \
    && apt-get -y autoremove \
    && rm -rf /var/lib/apt/lists/*

ARG NOVNC=/opt/noVNC

# Clone noVNC and websockify.
RUN mkdir -p ${NOVNC}/utils/websockify \
    && curl -sSfL 'https://github.com/novnc/noVNC/archive/v1.2.0.tar.gz' | sudo tar -zxf - -C ${NOVNC} --strip=1 \
    && curl -sSfL 'https://github.com/novnc/websockify/archive/v0.10.0.tar.gz' | sudo tar -zxf - -C ${NOVNC}/utils/websockify --strip=1 \
    && find ${NOVNC} -name ".git*" | sudo xargs rm -rf \
    && ln -s /usr/bin/python3 /usr/bin/python \
    && ln -s $NOVNC/vnc.html $NOVNC/index.html

EXPOSE 5901 6080

################################################################################
# Add predefined configuration
################################################################################

COPY build/config/localsudo       /etc/sudoers.d/
COPY build/run.sh                 /bin/run.sh
COPY build/utils.sh               /bin/utils.sh
COPY build/readmes/welcome_readme /etc/welcome_readme
COPY build/readmes/help_readme    /etc/help_readme
COPY build/readmes/vnc_readme     /etc/vnc_readme
COPY build/readmes/browser_readme /etc/browser_readme

# Set up the README files to show the correct MATLAB release
RUN for file in $(ls /etc/*_readme); do \
    sed -i "s/{{MATLAB_RELEASE}}/${MATLAB_RELEASE}/g" "${file}"; \
    done

################################################################################
# Setup matlab user
################################################################################

ARG HOME=/home/matlab

# Add "matlab" user, grant sudo permission and set default VNC password to "matlab".
RUN adduser --home ${HOME} --shell /bin/bash --disabled-password --gecos "" matlab \
    && echo "matlab ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/matlab \
    && chmod 0440 /etc/sudoers.d/matlab

# Set user and work directory.
USER matlab
WORKDIR /home/matlab

################################################################################
# Install MATLAB
################################################################################

# Run mpm to install MATLAB in the target location and delete the mpm installation afterwards.
# If mpm fails to install successfully, then print the logfile in the terminal, otherwise clean up.
# Remove the pre packaged libstdc libraries to force MATLAB to use the system libraries.
RUN wget -q https://www.mathworks.com/mpm/glnxa64/mpm \
    && chmod +x mpm \
    && sudo HOME=${HOME} ./mpm install \
    --release=${MATLAB_RELEASE} \
    --destination=${MATLAB_INSTALL_LOCATION} \
    ${MPM_ADDITIONAL_FLAGS} \
    --products=MATLAB Image_Processing_Toolbox Computer_Vision_Toolbox \
    || (echo "MPM Installation Failure. See below for more information:" && cat /tmp/mathworks_root.log && false) \
    && sudo rm -f ${MATLAB_INSTALL_LOCATION}/sys/os/glnxa64/libstdc++.* \
    && sudo rm -f mpm /tmp/mathworks_root.log \
    && sudo ln -s ${MATLAB_INSTALL_LOCATION}/bin/matlab /usr/local/bin/matlab


################################################################################
# Setup MathWorks Service Host
################################################################################

ENV MW_SERVICEHOST_USE_HOSTNAME_FOR_PERSISTENCE="false"

ARG MSH_TMP="/tmp/install_managed_msh.sh"
ARG MSH_ROOT="/opt/MathWorks/ServiceHost"

RUN wget -O "${MSH_TMP}" "${MATHWORKS_SERVICE_HOST_INSTALL_URL}" \
    && chmod +x "${MSH_TMP}" \
    && sudo ${MSH_TMP} --destination ${MSH_ROOT} \
    && rm "${MSH_TMP}"

################################################################################
# Setup matlab-proxy
################################################################################

RUN mkdir -p ${HOME}/Documents/MATLAB/Add-Ons \
    && touch ${HOME}/Documents/MATLAB/Add-Ons/.toolboxFolders \
    && touch ${HOME}/Documents/MATLAB/Add-Ons/.zipFolders \
    && python3 -m pipx install matlab-proxy

ENV PATH=${HOME}/.local/bin:${PATH}

# These lines disable token authentication for container workflows by default.
# To re-enable, unset this environment variable.
ENV MWI_ENABLE_TOKEN_AUTH="False"

ENV MWI_APP_PORT=8888
EXPOSE ${MWI_APP_PORT}

################################################################################
# Setup user experience data collection
################################################################################

# The following environment variable allows MathWorks to understand how this MathWorks
# product (MATLAB Dockerfile) is being used. This information helps us make MATLAB even better.
# Your content, and information about the content within your files, is not shared with MathWorks.
# To opt out of this service, unset the environment variables defined in the following line.
# To learn more, see the Help Make MATLAB Even Better section in the accompanying README:
# https://github.com/mathworks-ref-arch/matlab-dockerfile#help-make-matlab-even-better
ENV MW_DDUX_FORCE_ENABLE=true

################################################################################
# Final Setup
################################################################################

# Setup the container's desktop environment.
COPY build/config/xstartup       ${HOME}/.vnc/
COPY build/config/MATLAB.desktop ${HOME}/Desktop/
COPY build/config/xfce4/         ${HOME}/.config/xfce4/
COPY build/config/startup.m      ${HOME}/Documents/MATLAB/

# Make sure that MATLAB is running Windows shortcuts.
COPY build/config/matlab.prf     ${HOME}/.matlab/${MATLAB_RELEASE}/

# By adding this configuration file we turn off the screensaver for the VNC session by default.
COPY build/config/xscreensaver   ${HOME}/.xscreensaver

RUN sudo chown -R matlab:matlab ${HOME} \
    && chmod +x  ${HOME}/.vnc/xstartup \
    && chmod u+w ${HOME}/Desktop/MATLAB.desktop \
    && chmod u+w ${HOME}/.matlab/${MATLAB_RELEASE}/matlab.prf \
    && chmod +x  ${HOME}/Desktop/MATLAB.desktop \
    && touch     ${HOME}/.Xauthority \
    && sed -i "s/{{MATLAB_RELEASE}}/${MATLAB_RELEASE}/g" ${HOME}/Desktop/MATLAB.desktop

COPY build/config/novnc_login_redirect.html ${NOVNC}/redirect.html


ARG DEFAULT_VNC_PASSWORD=matlab

RUN printf "${DEFAULT_VNC_PASSWORD}\n${DEFAULT_VNC_PASSWORD}\n\n" | vncpasswd

WORKDIR /home/matlab/Documents/MATLAB/
ENTRYPOINT ["tini", "--", "/bin/run.sh"]
CMD []

################################################################################
# Build target for the mathworks/matlab-deep-learning image
################################################################################

FROM build AS matlab-deep-learning

ARG MATLAB_RELEASE
ARG MATLAB_INSTALL_LOCATION
ARG ADDITIONAL_PRODUCTS

# Run mpm to install MATLAB in the target location and delete the mpm installation afterwards.
# If mpm fails to install successfully, then print the logfile in the terminal, otherwise clean up.
# Pass in $HOME variable to install support packages into the user's HOME folder.
# Remove the pre packaged libstdc libraries to force MATLAB to use the system libraries.
RUN wget -q https://www.mathworks.com/mpm/glnxa64/mpm \
    && chmod +x mpm \
    && sudo HOME=${HOME} ./mpm install \
    --release=${MATLAB_RELEASE} \
    --destination=${MATLAB_INSTALL_LOCATION} \
    ${MPM_ADDITIONAL_FLAGS} \
    --products ${ADDITIONAL_PRODUCTS} \
    || (echo "MPM Installation Failure. See below for more information:" && cat /tmp/mathworks_root.log && false) \
    && sudo rm -f ${MATLAB_INSTALL_LOCATION}/sys/os/glnxa64/libstdc++.* \
    && sudo rm -rf mpm /tmp/mathworks_root.log

COPY build/config/matlab_deep_learning/MNISTExample.mlx ${HOME}/Documents/MATLAB/

# Set DDUX variables.
ENV MW_CONTEXT_TAGS=MATLAB:DEEPLEARNING:DOCKERHUB:V1
ENV MHLM_CONTEXT=MATLAB_DL_DOCKERHUB

################################################################################
# Build target for the mathworks/matlab image
################################################################################

FROM build AS matlab

# Set DDUX variables.
ENV MW_CONTEXT_TAGS=MATLAB:DOCKERHUB:V1
ENV MHLM_CONTEXT=MATLAB_DOCKERHUB

COPY App.m /home/matlab/
COPY .arkitekt_next /home/matlab/.arkitekt_next
COPY app.py /home/matlab/

RUN pip install arkitekt-next
