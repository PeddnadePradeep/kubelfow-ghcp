
#
# NOTE: Use the Makefiles to build this image correctly.
#

FROM ghcr.io/kubeflow/kubeflow/notebook-servers/base:sha-4a26c7b5e9575410613faf7df6735aa1883a2d24


USER root

RUN apt-get update && \
    apt-get install -y jq && \
    rm -rf /var/lib/apt/lists/*

ARG CODESERVER_VERSION=4.104.1
ARG TARGETARCH="amd64"

RUN curl -fsSL "https://github.com/coder/code-server/releases/download/v${CODESERVER_VERSION}/code-server_${CODESERVER_VERSION}_${TARGETARCH}.deb" -o /tmp/code-server.deb \
 && dpkg -i /tmp/code-server.deb \
 && rm -f /tmp/code-server.deb
COPY --chown=${NB_USER}:${NB_GID} --chmod=755 s6/ /etc

USER $NB_UID

EXPOSE 8888

USER root
# args - software versions
# https://open-vsx.org/extension/ms-python/python
# https://open-vsx.org/extension/ms-toolsai/jupyter
# https://github.com/ipython/ipykernel/releases
ARG CODESERVER_PYTHON_VERSION=2025.0.0
ARG CODESERVER_JUPYTER_VERSION=2024.11.0
ARG IPYKERNEL_VERSION=6.29.5
ARG MINIFORGE_VERSION=24.11.3-0
ARG PIP_VERSION=24.3.1
ARG PYTHON_VERSION=3.11.11




# setup environment for conda
ENV CONDA_DIR /opt/conda
ENV PATH "${CONDA_DIR}/bin:${PATH}"
RUN mkdir -pv ${CONDA_DIR} \
 && chmod 2775 ${CONDA_DIR} \
 && echo ". /opt/conda/etc/profile.d/conda.sh" >> ${HOME}/.bashrc \
 && echo ". /opt/conda/etc/profile.d/conda.sh" >> /etc/profile \
 && echo "conda activate base" >> ${HOME}/.bashrc \
 && echo "conda activate base" >> /etc/profile \
 && chown -R ${NB_USER}:${NB_GID} ${CONDA_DIR} \
 && chown -R ${NB_USER}:${USERS_GID} ${HOME}

USER $NB_UID

# install - conda, pip, python
RUN case "${TARGETARCH}" in \
      amd64) MINIFORGE_ARCH="x86_64" ;; \
      arm64) MINIFORGE_ARCH="aarch64" ;; \
      ppc64le) MINIFORGE_ARCH="ppc64le" ;; \
      *) echo "Unsupported architecture: ${TARGETARCH}"; exit 1 ;; \
    esac \
 && curl -fsSL "https://github.com/conda-forge/miniforge/releases/download/${MINIFORGE_VERSION}/Miniforge3-${MINIFORGE_VERSION}-Linux-${MINIFORGE_ARCH}.sh" -o /tmp/Miniforge3.sh \
 && curl -fsSL "https://github.com/conda-forge/miniforge/releases/download/${MINIFORGE_VERSION}/Miniforge3-${MINIFORGE_VERSION}-Linux-${MINIFORGE_ARCH}.sh.sha256" -o /tmp/Miniforge3.sh.sha256 \
 && echo "$(cat /tmp/Miniforge3.sh.sha256 | awk '{ print $1; }')  /tmp/Miniforge3.sh" | sha256sum -c - \
 && rm /tmp/Miniforge3.sh.sha256 \
 && /bin/bash /tmp/Miniforge3.sh -b -f -p ${CONDA_DIR} \
 && rm /tmp/Miniforge3.sh \
 && conda config --system --set auto_update_conda false \
 && conda config --system --set show_channel_urls true \
 && echo "python ==${PYTHON_VERSION}" >> ${CONDA_DIR}/conda-meta/pinned \
 && conda install -y -q \
    python=${PYTHON_VERSION} \
    pip=${PIP_VERSION} \
 && conda update -y -q --all \
 && conda clean -a -f -y

# install - ipykernel
# NOTE: we need this for jupyter codeserver extension to work
    # Pins the ipykernel package to the specified version in the conda environment,
    # ensuring that conda does not update it beyond the given version during future installs or updates.
RUN echo "ipykernel ==${IPYKERNEL_VERSION}" >> ${CONDA_DIR}/conda-meta/pinned \
 && conda install -y -q \
    ipykernel==${IPYKERNEL_VERSION} \
 && conda clean -a -f -y

# # install - requirements.txt
COPY --chown=${NB_USER}:${NB_GID} requirements.txt /tmp
RUN python3 -m pip install -r /tmp/requirements.txt --quiet --no-cache-dir \
 && rm -f /tmp/requirements.txt

#RUN curl -fsSL "https://marketplace.visualstudio.com/_apis/public/gallery/publishers/GitHub/vsextensions/copilotvs/1.206.0.0/vspackage" -o /tmp/GitHub.Copilot.Vsix.1.206.0.0.vsix \
#&& gunzip -c /tmp/GitHub.Copilot.Vsix.1.206.0.0.vsix > /tmp/github-copilot-decompressed.vsix \
#&& curl -fsSL "https://marketplace.visualstudio.com/_apis/public/gallery/publishers/VisualStudioExptTeam/vsextensions/VSGitHubCopilot/0.2.765.20217/vspackage" -o /tmp/VisualStudio.GitHub.Copilot.vsix \
#&& gunzip -c /tmp/VisualStudio.GitHub.Copilot.vsix > /tmp/github-copilot-chat-decompressed.vsix
# install - codeserver extensions
# RUN code-server --install-extension "ms-python.python@${CODESERVER_PYTHON_VERSION}" --force \
#  && code-server --install-extension "ms-toolsai.jupyter@${CODESERVER_JUPYTER_VERSION}" --force \
#  && code-server --list-extensions --show-versions

COPY --chown=${NB_USER}:${NB_GID} install-copilot.sh /tmp/install-copilot.sh
RUN chmod +x /tmp/install-copilot.sh \
  && /tmp/install-copilot.sh \
  && rm -f /tmp/install-copilot.sh \
  && code-server --list-extensions --show-versions


# # home - pre-populate home with files for this image
COPY --chown=${NB_USER}:${NB_GID} home/. ${HOME}/

# # s6 - 01-copy-tmp-home
# # NOTE: the contents of $HOME_TMP are copied to $HOME at runtime
# #       this is a workaround because a PVC will be mounted at $HOME
# #       and the contents of $HOME will be hidden
RUN cp -p -r -T "${HOME}" "${HOME_TMP}" \
    # give group same access as user (needed for OpenShift)
 && chmod -R g=u "${HOME_TMP}"


