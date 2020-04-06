# syntax = docker/dockerfile:experimental

ARG BASE_IMAGE=ubuntu:18.04
ARG OFFICIAL_IMAGE=pytorch/pytorch:1.4-cuda10.1-cudnn7-runtime

ARG NB_USER="jovyan"
ARG NB_UID="1000"
ARG NB_GID="100"


# load the official image as build source for the /opt/conda directory
FROM $OFFICIAL_IMAGE as source
ARG NB_USER
ARG NB_UID
ARG NB_GID
COPY fix-permissions /usr/local/bin/fix-permissions
RUN chmod a+rx /usr/local/bin/fix-permissions
RUN useradd -m -s /bin/bash -N -u $NB_UID $NB_USER
RUN fix-permissions /opt/conda


# Recreate the official pytorch image using fixed permissions
# We do this to keep the size as small as possible
# If we don't do it this way then the image will contain two copies
# of each conda file - the original and the updated permissions version.
FROM $BASE_IMAGE as official
LABEL com.nvidia.volumes.needed="nvidia_driver"

ARG NB_USER
ARG NB_UID
ARG NB_GID

ENV DEBIAN_FRONTEND noninteractive
RUN --mount=type=cache,id=apt-final,target=/var/cache/apt \
    apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates \
        libjpeg-dev \
        libpng-dev && \
    rm -rf /var/lib/apt/lists/*

COPY fix-permissions /usr/local/bin/fix-permissions
RUN chmod a+rx /usr/local/bin/fix-permissions

RUN sed -i 's/^#force_color_prompt=yes/force_color_prompt=yes/' /etc/skel/.bashrc && \
    useradd -m -s /bin/bash -N -u $NB_UID $NB_USER && \
    mkdir /opt/conda && chown $NB_USER:$NB_GID /opt/conda && \
    fix-permissions /home/$NB_USER

COPY --chown=$NB_USER:$NB_GID --from=source /opt/conda /opt/conda

ENV CONDA_DIR=/opt/conda \
    HOME=/home/$NB_USER

ENV PATH /opt/conda/bin:$PATH
ENV NVIDIA_VISIBLE_DEVICES all
ENV NVIDIA_DRIVER_CAPABILITIES compute,utility
ENV LD_LIBRARY_PATH /usr/local/nvidia/lib:/usr/local/nvidia/lib64

LABEL maintainer="ejsmit"

RUN --mount=type=cache,id=apt-final,target=/var/cache/apt \
    apt-get update && apt-get install -yq --no-install-recommends \
    sudo \
    locales \
    run-one \
    mc \
    git \
    tzdata \
    unzip \
    nano \
 && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN echo "en_GB.UTF-8 UTF-8" > /etc/locale.gen && \
    locale-gen && \
    echo "Europe/London"  | sudo tee /etc/timezone && \
    rm /etc/localtime && \
    dpkg-reconfigure --frontend noninteractive tzdata

# Create NB_USER wtih name jovyan user with UID=1000 and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    chown $NB_USER:$NB_GID $CONDA_DIR && \
    chmod g+w /etc/passwd && \
    fix-permissions $HOME && \
    fix-permissions $CONDA_DIR

USER $NB_UID
WORKDIR $HOME


# Install Tini
RUN conda install --quiet --yes 'tini=0.18.0' && \
    fix-permissions $CONDA_DIR && \
    fix-permissions /home/$NB_USER

RUN conda install --yes --quiet \
    nodejs \
    notebook \
    jupyterlab && \
    jupyter notebook --generate-config && \
    rm -rf $CONDA_DIR/share/jupyter/lab/staging && \
    fix-permissions $CONDA_DIR && \
    fix-permissions /home/$NB_USER

RUN conda install --quiet --yes \
   bokeh \
   h5py \
   hdf5 \
   matplotlib-base \
   numpy \
   pandas \
   scikit-image \
   scikit-learn \
   sympy \
   scipy \
   seaborn \
   pillow \
   ipywidgets \
   ipympl  && \
   jupyter labextension install @jupyter-widgets/jupyterlab-manager --no-build && \
   jupyter labextension install @bokeh/jupyter_bokeh --no-build && \
   jupyter labextension install jupyter-matplotlib --no-build && \
   jupyter lab build && \
   jupyter lab clean && \
   npm cache clean --force && \
   rm -rf /home/$NB_USER/.cache/yarn && \
   rm -rf /home/$NB_USER/.node-gyp && \
   fix-permissions $CONDA_DIR && \
   fix-permissions /home/$NB_USER


# Install facets which does not have a pip or conda package at the moment
RUN cd /tmp && \
   git clone https://github.com/PAIR-code/facets.git && \
   cd facets && \
   jupyter nbextension install facets-dist/ --sys-prefix && \
   cd && \
   rm -rf /tmp/facets && \
   fix-permissions $CONDA_DIR && \
   fix-permissions /home/$NB_USER


# Import matplotlib the first time to build the font cache.
ENV XDG_CACHE_HOME /home/$NB_USER/.cache/
RUN MPLBACKEND=Agg python -c "import matplotlib.pyplot" && \
   fix-permissions /home/$NB_USER


# Clean installation
RUN conda clean --all -f -y && \
   fix-permissions $CONDA_DIR && \
   fix-permissions /home/$NB_USER

# Copy local files as late as possible to avoid cache busting
COPY start.sh start-notebook.sh start-singleuser.sh /usr/local/bin/
COPY jupyter_notebook_config.py /etc/jupyter/

# Fix permissions on /etc/jupyter as root
USER root
RUN fix-permissions /etc/jupyter/
USER $NB_UID

EXPOSE 8888

# Configure container startup
ENTRYPOINT ["tini", "-g", "--"]
CMD ["start-notebook.sh"]
