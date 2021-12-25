FROM openjdk:11.0.13-slim-buster

LABEL maintainer="v.v.shinkevich@gmail.com"

# This Dockerfile adds a non-root user with sudo access. Use the "remoteUser"
# property in devcontainer.json to use it. On Linux, the container user's GID/UIDs
# will be updated to match your local UID/GID (when using the dockerFile property).
# See https://aka.ms/vscode-remote/containers/non-root-user for details.
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID

# Configure apt
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
    && apt-get -y install --no-install-recommends apt-utils dialog 2>&1 \
    #
    # Create a non-root user to use if preferred - see https://aka.ms/vscode-remote/containers/non-root-user.
    && groupadd --gid $USER_GID $USERNAME \
    && useradd -s /bin/bash --uid $USER_UID --gid $USER_GID -m $USERNAME \
    # [Optional] Add sudo support for the non-root user
    && apt-get install -y sudo \
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME\
    && chmod 0440 /etc/sudoers.d/$USERNAME \
    #
    # Verify git, needed tools installed
    && apt-get -y install git openssh-client less iproute2 procps curl lsb-release


#------------------------------------------------Install Python-----------------------------------------------
# ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:$PATH
ENV LANG C.UTF-8

# runtime dependencies
RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		ca-certificates \
		netbase \
		tzdata \
	; \
	rm -rf /var/lib/apt/lists/*

ENV GPG_KEY E3FF2839C048B25C084DEBE9B26995E310250568
ENV PYTHON_VERSION 3.9.9

RUN set -ex \
	\
	&& savedAptMark="$(apt-mark showmanual)" \
	&& apt-get update && apt-get install -y --no-install-recommends \
		dpkg-dev \
		gcc \
		libbluetooth-dev \
		libbz2-dev \
		libc6-dev \
		libexpat1-dev \
		libffi-dev \
		libgdbm-dev \
		liblzma-dev \
		libncursesw5-dev \
		libreadline-dev \
		libsqlite3-dev \
		libssl-dev \
		make \
		tk-dev \
		uuid-dev \
		wget \
		xz-utils \
		zlib1g-dev \
# as of Stretch, "gpg" is no longer included by default
		$(command -v gpg > /dev/null || echo 'gnupg dirmngr') \
	\
	&& wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz" \
	&& wget -O python.tar.xz.asc "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz.asc" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$GPG_KEY" \
	&& gpg --batch --verify python.tar.xz.asc python.tar.xz \
	&& { command -v gpgconf > /dev/null && gpgconf --kill all || :; } \
	&& rm -rf "$GNUPGHOME" python.tar.xz.asc \
	&& mkdir -p /usr/src/python \
	&& tar -xJC /usr/src/python --strip-components=1 -f python.tar.xz \
	&& rm python.tar.xz \
	\
	&& cd /usr/src/python \
	&& gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
	&& ./configure \
		--build="$gnuArch" \
		--enable-loadable-sqlite-extensions \
		--enable-optimizations \
		--enable-option-checking=fatal \
		--enable-shared \
		--with-system-expat \
		--with-system-ffi \
		--without-ensurepip \
	&& make -j "$(nproc)" \
		LDFLAGS="-Wl,--strip-all" \
	&& make install \
	&& rm -rf /usr/src/python \
	\
	&& find /usr/local -depth \
		\( \
			\( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
			-o \( -type f -a \( -name '*.pyc' -o -name '*.pyo' -o -name '*.a' \) \) \
		\) -exec rm -rf '{}' + \
	\
	&& ldconfig \
	\
	&& apt-mark auto '.*' > /dev/null \
	&& apt-mark manual $savedAptMark \
	&& find /usr/local -type f -executable -not \( -name '*tkinter*' \) -exec ldd '{}' ';' \
		| awk '/=>/ { print $(NF-1) }' \
		| sort -u \
		| xargs -r dpkg-query --search \
		| cut -d: -f1 \
		| sort -u \
		| xargs -r apt-mark manual \
	&& apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
	&& rm -rf /var/lib/apt/lists/* \
	\
	&& python3 --version

# make some useful symlinks that are expected to exist
RUN cd /usr/local/bin \
	&& ln -s idle3 idle \
	&& ln -s pydoc3 pydoc \
	&& ln -s python3 python \
	&& ln -s python3-config python-config

RUN set -ex; \
	\
	savedAptMark="$(apt-mark showmanual)"; \
	apt-get update; \
	apt-get install -y --no-install-recommends curl; \
	\
	curl -sSL https://bootstrap.pypa.io/get-pip.py -o get-pip.py; \
	\
	apt-mark auto '.*' > /dev/null; \
	[ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	rm -rf /var/lib/apt/lists/*; \
	\
	python get-pip.py \
		--disable-pip-version-check \
		--no-cache-dir \
	; \
	pip --version; \
	\
	find /usr/local -depth \
		\( \
			\( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
			-o \
			\( -type f -a \( -name '*.pyc' -o -name '*.pyo' \) \) \
		\) -exec rm -rf '{}' +; \
	rm -f get-pip.py                   

RUN pip3 install pandas scikit-learn
#-------------------------------------------------------------------------------------------------------------

#-------------------------------------------Install Scala-----------------------------------------------------
ENV SCALA_VERSION 2.12.10
ENV SCALA_HOME /usr/local/scala
ENV PATH $PATH:$SCALA_HOME/bin

RUN apt-get update && apt-get install --assume-yes wget
RUN wget "https://downloads.typesafe.com/scala/${SCALA_VERSION}/scala-${SCALA_VERSION}.tgz" \
    && tar xzf "scala-${SCALA_VERSION}.tgz" \
    && rm "scala-${SCALA_VERSION}/bin/"*.bat \
    && mkdir "${SCALA_HOME}" \
    && mv "scala-${SCALA_VERSION}/bin" "scala-${SCALA_VERSION}/lib" "${SCALA_HOME}" \
    && ln -s "${SCALA_HOME}/bin/"* "/usr/bin/" \
    && rm -rf "scala-${SCALA_VERSION}" \
    && rm "scala-${SCALA_VERSION}.tgz" \
    && echo -ne "- with scala $SCALA_VERSION\n" >> /root/.built \
    && scala -version
#-------------------------------------------------------------------------------------------------------------

#--------------------------------Uncomment the following steps to install SBT---------------------------------
ENV SBT_VERSION 1.5.8
ENV SBT_HOME /usr/local/sbt
ENV PATH $PATH:$SBT_HOME/bin

RUN apt-get update && apt-get install --assume-yes curl
RUN curl -sL "https://github.com/sbt/sbt/releases/download/v$SBT_VERSION/sbt-$SBT_VERSION.tgz" | gunzip | tar -x -C /usr/local \
    && echo -ne "- with sbt $SBT_VERSION\n" >> /root/.built
#-------------------------------------------------------------------------------------------------------------

#------------------------------------Install Apache Spark-----------------------------------------------------
ARG SPARK_VERSION=3.1.2

RUN mkdir -p /opt/spark \
    && curl -fsSL -o /tmp/spark.tgz https://archive.apache.org/dist/spark/spark-${SPARK_VERSION}/spark-${SPARK_VERSION}-bin-hadoop3.2.tgz \
    && tar -xzf /tmp/spark.tgz -C /opt/spark --strip-components=1 \
    && rm -f /tmp/spark.tgz \
    && echo -ne "- with spark $SPARK_VERSION\n" >> /root/.built 
  
ENV SPARK_HOME /opt/spark
ENV SPARK_DIST_CLASSPATH $HADOOP_HOME/etc/hadoop:$HADOOP_HOME/share/hadoop/common/lib/*:$HADOOP_HOME/share/hadoop/common/*:$HADOOP_HOME/share/hadoop/hdfs:$HADOOP_HOME/share/hadoop/hdfs/lib/*:$HADOOP_HOME/share/hadoop/hdfs/*:$HADOOP_HOME/share/hadoop/yarn/lib/*:$HADOOP_HOME/share/hadoop/yarn/*:$HADOOP_HOME/share/hadoop/mapreduce/lib/*:$HADOOP_HOME/share/hadoop/mapreduce/*:$HADOOP_HOME/contrib/capacity-scheduler/*.jar
ENV PATH $PATH:$SPARK_HOME/bin
#-------------------------------------------------------------------------------------------------------------

#-------------------Uncomment the following steps to install Maven CLI Tools----------------------------------
# ARG MAVEN_VERSION=3.6.3
# ARG MAVEN_SHA=c35a1803a6e70a126e80b2b3ae33eed961f83ed74d18fcd16909b2d44d7dada3203f1ffe726c17ef8dcca2dcaa9fca676987befeadc9b9f759967a8cb77181c0
# RUN mkdir -p /usr/share/maven /usr/share/maven/ref \
#   && curl -fsSL -o /tmp/apache-maven.tar.gz https://archive.apache.org/dist/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
#   && echo "${MAVEN_SHA} /tmp/apache-maven.tar.gz" | sha512sum -c - \
#   && tar -xzf /tmp/apache-maven.tar.gz -C /usr/share/maven --strip-components=1 \
#   && rm -f /tmp/apache-maven.tar.gz \
#   && ln -s /usr/share/maven/bin/mvn /usr/bin/mvn
# COPY maven-settings.xml /usr/share/maven/ref/
# ENV MAVEN_HOME /usr/share/maven
# ENV MAVEN_CONFIG /root/.m2
#-------------------------------------------------------------------------------------------------------------

#-------------------Uncomment the following steps to install Gradle CLI Tools---------------------------------
# ENV GRADLE_HOME /opt/gradle
# ENV GRADLE_VERSION 5.4.1
# ARG GRADLE_DOWNLOAD_SHA256=7bdbad1e4f54f13c8a78abc00c26d44dd8709d4aedb704d913fb1bb78ac025dc
# RUN curl -sSL --output gradle.zip "https://services.gradle.org/distributions/gradle-${GRADLE_VERSION}-bin.zip" \
#     && echo "${GRADLE_DOWNLOAD_SHA256} *gradle.zip" | sha256sum --check - \
#     && unzip gradle.zip \
#     && rm gradle.zip \
#     && mv "gradle-${GRADLE_VERSION}" "${GRADLE_HOME}/" \
#     && ln -s "${GRADLE_HOME}/bin/gradle" /usr/bin/gradle
#-------------------------------------------------------------------------------------------------------------

# Clean up
RUN apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*
ENV DEBIAN_FRONTEND=dialog

# Allow for a consistant java home location for settings - image is changing over time
RUN if [ ! -d "/docker-java-home" ]; then ln -s "${JAVA_HOME}" /docker-java-home; fi
