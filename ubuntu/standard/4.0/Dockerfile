# Copyright 2019-2022 Amazon.com, Inc. or its affiliates. All Rights Reserved.
#
# Licensed under the Amazon Software License (the "License"). You may not use this file except in compliance with the License.
# A copy of the License is located at
#
#    http://aws.amazon.com/asl/
#
# or in the "license" file accompanying this file.
# This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied.
# See the License for the specific language governing permissions and limitations under the License.
FROM public.ecr.aws/ubuntu/ubuntu:18.04 AS core

ENV DEBIAN_FRONTEND="noninteractive"

# Install git, SSH, and other utilities
RUN set -ex \
    && echo 'Acquire::CompressionTypes::Order:: "gz";' > /etc/apt/apt.conf.d/99use-gzip-compression \
    && apt-get update \
    && apt install -y -qq apt-transport-https gnupg ca-certificates \
    && apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF \
    && echo "deb https://download.mono-project.com/repo/ubuntu stable-bionic main" | tee /etc/apt/sources.list.d/mono-official-stable.list \
    && apt-get install software-properties-common -y -qq --no-install-recommends \
    && apt-add-repository -y ppa:git-core/ppa \
    && apt-get update \
    && apt-get install git=1:2.* -y -qq --no-install-recommends \
    && git version \
    && apt-get install -y -qq --no-install-recommends openssh-client \
    && mkdir ~/.ssh \
    && mkdir -p /opt/tools \
    && mkdir -p /codebuild/image/config \
    && touch ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa -H github.com >> ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa -H bitbucket.org >> ~/.ssh/known_hosts \
    && chmod 600 ~/.ssh/known_hosts \
    && apt-get install -y -qq --no-install-recommends \
          apt-utils asciidoc autoconf automake build-essential bzip2 \
          bzr curl cvs cvsps dirmngr docbook-xml docbook-xsl dpkg-dev \
          e2fsprogs expect fakeroot file g++ gcc gettext gettext-base \
          groff gzip imagemagick iptables jq less libapr1 libaprutil1 \
          libargon2-0-dev libbz2-dev libc6-dev libcurl4-openssl-dev \
          libdb-dev libdbd-sqlite3-perl libdbi-perl libdpkg-perl \
          libedit-dev liberror-perl libevent-dev libffi-dev libgeoip-dev \
          libglib2.0-dev libhttp-date-perl libio-pty-perl libjpeg-dev \
          libkrb5-dev liblzma-dev libmagickcore-dev libmagickwand-dev \
          libmysqlclient-dev libncurses5-dev libncursesw5-dev libonig-dev \
          libpq-dev libreadline-dev libserf-1-1 libsqlite3-dev libssl-dev \
          libsvn1 libsvn-perl libtcl8.6 libtidy-dev libtimedate-perl \
          libtool libwebp-dev libxml2-dev libxml2-utils libxslt1-dev \
          libyaml-dev libyaml-perl llvm locales make mercurial mlocate mono-devel \
          netbase openssl patch pkg-config procps python-bzrlib \
          python-configobj python-openssl rsync sgml-base sgml-data subversion \
          tar tcl tcl8.6 tk tk-dev unzip wget xfsprogs xml-core xmlto xsltproc \
          libzip4 libzip-dev vim xvfb xz-utils zip zlib1g-dev \
    && rm -rf /var/lib/apt/lists/* 

RUN useradd codebuild-user

#=======================End of layer: core  =================


FROM core AS tools

# Install Firefox
RUN set -ex \
    && apt-add-repository -y "deb http://archive.canonical.com/ubuntu $(lsb_release -sc) partner" \
    && apt-add-repository -y ppa:malteworld/ppa && apt-get update \
    && apt-get install -y -qq libgtk-3-0 libglib2.0-0 libdbus-glib-1-2 libdbus-1-3 libasound2 \
    && wget -qO ~/FirefoxSetup.tar.bz2 "https://download.mozilla.org/?product=firefox-latest&os=linux64" \
    && tar xjf ~/FirefoxSetup.tar.bz2 -C /opt/ \
    && ln -s /opt/firefox/firefox /usr/local/bin/firefox \
    && rm ~/FirefoxSetup.tar.bz2 \
    && firefox --version

# Install GeckoDriver
RUN set -ex \
    && curl -s https://api.github.com/repos/mozilla/geckodriver/releases/latest | jq -r '.assets[] | select(.browser_download_url | endswith("linux64.tar.gz")).browser_download_url' | wget -qO /tmp/geckodriver-latest-linux64.tar.gz -qi - \
    && tar -xzf /tmp/geckodriver-latest-linux64.tar.gz -C /opt \
    && rm /tmp/geckodriver-latest-linux64.tar.gz \
    && chmod 755 /opt/geckodriver \
    && ln -s /opt/geckodriver /usr/bin/geckodriver \
    && geckodriver --version

# Install Chrome
RUN set -ex \
    && curl --silent --show-error --location --fail --retry 3 --output /tmp/google-chrome-stable_current_amd64.deb https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb \
    && (dpkg -i /tmp/google-chrome-stable_current_amd64.deb || apt-get -fy install) \
    && rm -rf /tmp/google-chrome-stable_current_amd64.deb \
    && sed -i 's|HERE/chrome"|HERE/chrome" --disable-setuid-sandbox --no-sandbox|g' "/opt/google/chrome/google-chrome" \
    && google-chrome --version

# Install ChromeDriver
RUN set -ex \
    && CHROME_VERSION=`google-chrome --version | awk -F '[ .]' '{print $3"."$4"."$5}'` \
    && CHROME_DRIVER_VERSION=`wget -qO- chromedriver.storage.googleapis.com/LATEST_RELEASE_$CHROME_VERSION` \
    && wget -qO /tmp/chromedriver_linux64.zip https://chromedriver.storage.googleapis.com/$CHROME_DRIVER_VERSION/chromedriver_linux64.zip \
    && unzip -q /tmp/chromedriver_linux64.zip -d /opt \
    && rm /tmp/chromedriver_linux64.zip \
    && mv /opt/chromedriver /opt/chromedriver-$CHROME_DRIVER_VERSION \
    && chmod 755 /opt/chromedriver-$CHROME_DRIVER_VERSION \
    && ln -s /opt/chromedriver-$CHROME_DRIVER_VERSION /usr/bin/chromedriver \
    && chromedriver --version

# Install stunnel
RUN set -ex \
   && STUNNEL_VERSION=5.67 \
   && STUNNEL_TAR=stunnel-$STUNNEL_VERSION.tar.gz \
   && STUNNEL_SHA256="3086939ee6407516c59b0ba3fbf555338f9d52f459bcab6337c0f00e91ea8456" \
   && curl -o $STUNNEL_TAR https://www.usenix.org.uk/mirrors/stunnel/archive/5.x/$STUNNEL_TAR && echo "$STUNNEL_SHA256 $STUNNEL_TAR" | sha256sum --check && tar xfz $STUNNEL_TAR \
   && cd stunnel-$STUNNEL_VERSION \
   && ./configure \
   && make -j4 \
   && make install \
   && openssl genrsa -out key.pem 2048 \
   && openssl req -new -x509 -key key.pem -out cert.pem -days 1095 -subj "/C=US/ST=Washington/L=Seattle/O=Amazon/OU=Codebuild/CN=codebuild.amazon.com" \
   && cat key.pem cert.pem >> /usr/local/etc/stunnel/stunnel.pem \
   && cd .. && rm -rf stunnel-${STUNNEL_VERSION}*

# AWS Tools
# https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html
RUN curl -sS -o /usr/local/bin/aws-iam-authenticator https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/aws-iam-authenticator \
    && curl -sS -o /usr/local/bin/kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl \
    && curl -sS -o /usr/local/bin/ecs-cli https://s3.amazonaws.com/amazon-ecs-cli/ecs-cli-linux-amd64-latest \
    && curl -sS -L https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz | tar xz -C /usr/local/bin \
    && chmod +x /usr/local/bin/kubectl /usr/local/bin/aws-iam-authenticator /usr/local/bin/ecs-cli /usr/local/bin/eksctl

# Configure SSM
RUN set -ex \
    && mkdir /tmp/ssm \
    && cd /tmp/ssm \
    && wget -q https://s3.amazonaws.com/amazon-ssm-us-east-1/3.1.1374.0/debian_amd64/amazon-ssm-agent.deb \
    && dpkg -i amazon-ssm-agent.deb

# Install env tools for runtimes
# Dotnet
ENV PATH "/root/.dotnet/:/root/.dotnet/tools/:$PATH"
RUN set -ex  \
&& wget -qO /usr/local/bin/dotnet-install.sh https://dot.net/v1/dotnet-install.sh \
&& chmod +x /usr/local/bin/dotnet-install.sh 

#nodejs
ENV SRC_DIR="/usr/src"
ENV N_SRC_DIR="$SRC_DIR/n"
RUN git clone https://github.com/tj/n $N_SRC_DIR \
     && cd $N_SRC_DIR && make install 

#ruby
ENV RBENV_SRC_DIR="/usr/local/rbenv"

ENV PATH="/root/.rbenv/shims:$RBENV_SRC_DIR/bin:$RBENV_SRC_DIR/shims:$PATH" \
    RUBY_BUILD_SRC_DIR="$RBENV_SRC_DIR/plugins/ruby-build"

RUN set -ex \
    && git clone https://github.com/rbenv/rbenv.git $RBENV_SRC_DIR \
    && mkdir -p $RBENV_SRC_DIR/plugins \
    && git clone https://github.com/rbenv/ruby-build.git $RUBY_BUILD_SRC_DIR \
    && sh $RUBY_BUILD_SRC_DIR/install.sh

#python
RUN curl https://pyenv.run | bash
ENV PATH="/root/.pyenv/shims:/root/.pyenv/bin:$PATH"

#php
RUN curl -L https://raw.githubusercontent.com/phpenv/phpenv-installer/master/bin/phpenv-installer | bash
ENV PATH="/root/.phpenv/shims:/root/.phpenv/bin:$PATH"

#go
RUN git clone https://github.com/syndbg/goenv.git $HOME/.goenv
ENV PATH="/root/.goenv/shims:/root/.goenv/bin:/go/bin:$PATH"
ENV GOENV_DISABLE_GOPATH=1
ENV GOPATH="/go"

#=======================End of layer: tools  =================
FROM tools AS runtimes

#****************     .NET-CORE     *******************************************************

ENV DOTNET_31_SDK_VERSION="3.1.426"
ENV DOTNET_ROOT="/root/.dotnet"

# Add .NET Core Global Tools install folder to PATH
RUN  /usr/local/bin/dotnet-install.sh -v $DOTNET_31_SDK_VERSION \
     && dotnet --list-sdks \
     && rm -rf /tmp/*

# Trigger the population of the local package cache
ENV NUGET_XMLDOC_MODE skip
RUN set -ex \
    && mkdir warmup \
    && cd warmup \
    && dotnet new \
    && cd .. \
    && rm -rf warmup \
    && rm -rf /tmp/NuGetScratch

# Install GitVersion
ENV GITVERSION_VERSION="5.3.7"
RUN set -ex \
    && dotnet tool install --global GitVersion.Tool --version $GITVERSION_VERSION \
    && ln -s ~/.dotnet/tools/dotnet-gitversion /usr/local/bin/gitversion

# Install Powershell Core
# See instructions at https://docs.microsoft.com/en-us/powershell/scripting/setup/installing-powershell-core-on-linux
ENV POWERSHELL_VERSION 6.2.6
ENV POWERSHELL_DOWNLOAD_URL https://github.com/PowerShell/PowerShell/releases/download/v$POWERSHELL_VERSION/powershell-$POWERSHELL_VERSION-linux-x64.tar.gz
ENV POWERSHELL_DOWNLOAD_SHA ee5512d869ab9bd59bf17f417ff93013e0a169db91cf848ba2570d4818e05e17

RUN set -ex \
    && curl -SL $POWERSHELL_DOWNLOAD_URL --output powershell.tar.gz \
    && echo "$POWERSHELL_DOWNLOAD_SHA powershell.tar.gz" | sha256sum -c - \
    && mkdir -p /opt/microsoft/powershell/$POWERSHELL_VERSION \
    && tar zxf powershell.tar.gz -C /opt/microsoft/powershell/$POWERSHELL_VERSION \
    && rm powershell.tar.gz \
    && ln -s /opt/microsoft/powershell/$POWERSHELL_VERSION/pwsh /usr/bin/pwsh
#****************     END .NET-CORE     *******************************************************


#****************      NODEJS     ****************************************************

ENV NODE_12_VERSION="12.22.12" \
    NODE_10_VERSION="10.24.1"

RUN     n $NODE_10_VERSION && npm install --save-dev -g -f grunt && npm install --save-dev -g -f grunt-cli && npm install --save-dev -g -f webpack \
     && n $NODE_12_VERSION && npm install --save-dev -g -f grunt && npm install --save-dev -g -f grunt-cli && npm install --save-dev -g -f webpack \
     && curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
     && echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
     && apt-get update && apt-get install -y -qq --no-install-recommends yarn \
     && yarn --version \
     && cd / && rm -rf $N_SRC_DIR && rm -rf /tmp/*

#****************      END NODEJS     ****************************************************

#**************** RUBY *********************************************************

ENV RUBY_26_VERSION="2.6.10" \
    RUBY_27_VERSION="2.7.7"

RUN rbenv install $RUBY_26_VERSION && rm -rf /tmp/*
RUN rbenv install $RUBY_27_VERSION && rm -rf /tmp/* && rbenv global $RUBY_27_VERSION && ruby -v

#**************** END RUBY *****************************************************

#**************** PYTHON *****************************************************
ENV PYTHON_37_VERSION="3.7.16"
ENV PYTHON_38_VERSION="3.8.16"
ENV PYTHON_39_VERSION="3.9.16"

ENV PYTHON_PIP_VERSION=21.1.2
ENV PYYAML_VERSION=5.4.1

COPY tools/runtime_configs/python/$PYTHON_37_VERSION /root/.pyenv/plugins/python-build/share/python-build/$PYTHON_37_VERSION
RUN   env PYTHON_CONFIGURE_OPTS="--enable-shared" pyenv install $PYTHON_37_VERSION && rm -rf /tmp/*
RUN   pyenv global  $PYTHON_37_VERSION
RUN set -ex \
    && pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
    && pip3 install --no-cache-dir --upgrade "PyYAML==$PYYAML_VERSION" \
    && pip3 install --no-cache-dir --upgrade 'setuptools==57.4.0' wheel aws-sam-cli awscli boto3 pipenv virtualenv --use-feature=2020-resolver

COPY tools/runtime_configs/python/$PYTHON_38_VERSION /root/.pyenv/plugins/python-build/share/python-build/$PYTHON_38_VERSION
RUN   env PYTHON_CONFIGURE_OPTS="--enable-shared" pyenv install $PYTHON_38_VERSION && rm -rf /tmp/*
RUN   pyenv global  $PYTHON_38_VERSION
RUN set -ex \
    && pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
    && pip3 install --no-cache-dir --upgrade "PyYAML==$PYYAML_VERSION" \
    && pip3 install --no-cache-dir --upgrade 'setuptools==57.4.0' wheel aws-sam-cli awscli boto3 pipenv virtualenv --use-feature=2020-resolver

COPY tools/runtime_configs/python/$PYTHON_39_VERSION /root/.pyenv/plugins/python-build/share/python-build/$PYTHON_39_VERSION
RUN   env PYTHON_CONFIGURE_OPTS="--enable-shared" pyenv install $PYTHON_39_VERSION && rm -rf /tmp/*
RUN   pyenv global  $PYTHON_39_VERSION
RUN set -ex \
    && pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
    && pip3 install --no-cache-dir --upgrade "PyYAML==$PYYAML_VERSION" \
    && pip3 install --no-cache-dir --upgrade 'setuptools==57.4.0' wheel aws-sam-cli awscli boto3 pipenv virtualenv --use-feature=2020-resolver

#**************** END PYTHON *****************************************************

#****************      PHP     ****************************************************
ENV PHP_73_VERSION="7.3.33"
ENV PHP_74_VERSION="7.4.33"

RUN curl -L https://raw.githubusercontent.com/phpenv/phpenv-installer/master/bin/phpenv-installer | bash
ENV PATH="/root/.phpenv/shims:/root/.phpenv/bin:$PATH"
COPY tools/runtime_configs/php/$PHP_73_VERSION /root/.phpenv/plugins/php-build/share/php-build/definitions/$PHP_73_VERSION
RUN phpenv install $PHP_73_VERSION && rm -rf /tmp/* && phpenv global $PHP_73_VERSION
RUN echo "memory_limit = 1G;" >> "/root/.phpenv/versions/$PHP_73_VERSION/etc/conf.d/memory.ini"

COPY tools/runtime_configs/php/$PHP_74_VERSION /root/.phpenv/plugins/php-build/share/php-build/definitions/$PHP_74_VERSION
RUN phpenv install $PHP_74_VERSION && rm -rf /tmp/* && phpenv global $PHP_74_VERSION
RUN echo "memory_limit = 1G;" >> "/root/.phpenv/versions/$PHP_74_VERSION/etc/conf.d/memory.ini"

# Install Composer globally
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin --filename=composer
#****************      END PHP     ****************************************************

#****************     GOLANG     ****************************************************
ENV GOLANG_13_VERSION="1.13.15" \
    GOLANG_12_VERSION="1.12.17"

RUN goenv install $GOLANG_12_VERSION && rm -rf /tmp/*

RUN goenv install $GOLANG_13_VERSION && rm -rf /tmp/* && goenv global  $GOLANG_13_VERSION

RUN go get -u github.com/golang/dep/cmd/dep
#****************      END GOLANG     *******************************

#=======================End of layer: runtimes  =================

FROM runtimes AS runtimes_n_corretto

#****************      JAVA     ****************************************************
COPY tools/android-accept-licenses.sh /opt/tools/android-accept-licenses.sh

ENV JAVA_11_HOME="/usr/lib/jvm/java-11-amazon-corretto" \
    JDK_11_HOME="/usr/lib/jvm/java-11-amazon-corretto" \
    JRE_11_HOME="/usr/lib/jvm/java-11-amazon-corretto" \
    JAVA_8_HOME="/usr/lib/jvm/java-1.8.0-amazon-corretto" \
    JDK_8_HOME="/usr/lib/jvm/java-1.8.0-amazon-corretto" \
    JRE_8_HOME="/usr/lib/jvm/java-1.8.0-amazon-corretto/jre" \
    ANT_VERSION=1.10.12 \
    MAVEN_HOME="/opt/maven" \
    MAVEN_VERSION=3.6.3 \
    INSTALLED_GRADLE_VERSIONS="4.10.3 5.6.4" \
    GRADLE_VERSION=5.6.4 \
    SBT_VERSION=1.6.2 \
    ANDROID_HOME="/usr/local/android-sdk-linux" \
    GRADLE_PATH="$SRC_DIR/gradle" \
    ANDROID_SDK_MANAGER_VER="4333796" \
    ANDROID_SDK_BUILD_TOOLS="build-tools;29.0.3" \
    ANDROID_SDK_PLATFORM_TOOLS="platforms;android-29" \
    ANDROID_SDK_BUILD_TOOLS_28="build-tools;28.0.3" \
    ANDROID_SDK_PLATFORM_TOOLS_28="platforms;android-28" \
    ANDROID_SDK_EXTRAS="extras;android;m2repository extras;google;m2repository extras;google;google_play_services" \
    ANT_DOWNLOAD_SHA512="2287dc5cfc21043c14e5413f9afb1c87c9f266ec2a9ba2d3bf2285446f6e4ccb59b558bf2e5c57911a05dfa293c7d5c7ad60ac9f744ba11406f4e6f9a27b2403" \
    MAVEN_DOWNLOAD_SHA512="c35a1803a6e70a126e80b2b3ae33eed961f83ed74d18fcd16909b2d44d7dada3203f1ffe726c17ef8dcca2dcaa9fca676987befeadc9b9f759967a8cb77181c0" \
    GRADLE_DOWNLOADS_SHA256="abc10bcedb58806e8654210f96031db541bcd2d6fc3161e81cb0572d6a15e821 5.6.4\n336b6898b491f6334502d8074a6b8c2d73ed83b92123106bd4bf837f04111043 4.10.3" \
    ANDROID_SDK_MANAGER_SHA256="92ffee5a1d98d856634e8b71132e8a95d96c83a63fde1099be3d86df3106def9" \
    SBT_DOWNLOAD_SHA256="637637b6c4e6fa04ab62cd364061e32b12480b09001cd23303df62b36fadd440" \
    LOG4J_UNSAFE_VERSIONS="2.11.1 1.2.8"

ARG MAVEN_CONFIG_HOME="/root/.m2" 

ENV JAVA_HOME="$JAVA_11_HOME" \
    JDK_HOME="$JDK_11_HOME" \
    JRE_HOME="$JRE_11_HOME"

ENV PATH="${PATH}:/opt/tools:${ANDROID_HOME}/tools:${ANDROID_HOME}/tools/bin:${ANDROID_HOME}/platform-tools"

RUN set -ex \
    && apt-get update \
    && apt-get install -y -qq software-properties-common apt-utils \
    # Install Corretto 8
    && wget -qO- https://apt.corretto.aws/corretto.key | apt-key add - \
    && add-apt-repository 'deb https://apt.corretto.aws stable main' \
    && apt-get update \
    && apt-get install -y -qq java-1.8.0-amazon-corretto-jdk \
    && apt-get install -y -qq --no-install-recommends ca-certificates-java \
    # Ensure Java cacerts symlink points to valid location
    && update-ca-certificates -f \
    && dpkg --add-architecture i386 \
    && apt-get update \
    && apt-get install -y -qq --force-yes libc6-i386 \
       lib32stdc++6 lib32gcc1 lib32ncurses5 \
       lib32z1 libqt5widgets5 

RUN set -ex \
    # Install Android SDK manager
    && wget -q "https://dl.google.com/android/repository/sdk-tools-linux-${ANDROID_SDK_MANAGER_VER}.zip" -O /tmp/android-sdkmanager.zip \
    && echo "${ANDROID_SDK_MANAGER_SHA256} /tmp/android-sdkmanager.zip" | sha256sum -c - \
    && mkdir -p ${ANDROID_HOME} \
    && unzip -q /tmp/android-sdkmanager.zip -d ${ANDROID_HOME} \
    && rm /tmp/android-sdkmanager.zip \
    && chown -R root.root ${ANDROID_HOME} \
    && ln -s ${ANDROID_HOME}/tools/android /usr/bin/android \
    # Install Android
    && android-accept-licenses.sh "env JAVA_HOME=$JAVA_8_HOME JRE_HOME=$JRE_8_HOME JDK_HOME=$JDK_8_HOME sdkmanager --verbose platform-tools ${ANDROID_SDK_BUILD_TOOLS} ${ANDROID_SDK_PLATFORM_TOOLS} ${ANDROID_SDK_EXTRAS} ${ANDROID_SDK_NDK_TOOLS}" | grep -v '\[=' \
    && android-accept-licenses.sh "env JAVA_HOME=$JAVA_8_HOME JRE_HOME=$JRE_8_HOME JDK_HOME=$JDK_8_HOME sdkmanager --verbose platform-tools ${ANDROID_SDK_BUILD_TOOLS_28} ${ANDROID_SDK_PLATFORM_TOOLS_28}" | grep -v '\[=' \
    && android-accept-licenses.sh "env JAVA_HOME=$JAVA_8_HOME JRE_HOME=$JRE_8_HOME JDK_HOME=$JDK_8_HOME sdkmanager --licenses" \
    && apt-get install -y -qq python-setuptools

RUN set -ex \
    # Install Corretto 11
    # Note: We will use update-alternatives to make sure JDK11 has higher priority for all the tools
    && apt-get install -y -qq java-11-amazon-corretto-jdk \
    && for tool_path in $JAVA_HOME/bin/*; do \
          tool=`basename $tool_path`; \
          update-alternatives --install /usr/bin/$tool $tool $tool_path 10000; \
          update-alternatives --set $tool $tool_path; \
        done \
     && rm $JAVA_HOME/lib/security/cacerts && ln -s /etc/ssl/certs/java/cacerts $JAVA_HOME/lib/security/cacerts \
    # Install Ant
    && curl -LSso /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz https://archive.apache.org/dist/ant/binaries/apache-ant-$ANT_VERSION-bin.tar.gz  \
    && echo "$ANT_DOWNLOAD_SHA512 /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz" | sha512sum -c - \
    && tar -xzf /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz -C /opt \
    && rm /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz \
    && update-alternatives --install /usr/bin/ant ant /opt/apache-ant-$ANT_VERSION/bin/ant 10000 

RUN set -ex \
    # Install Maven
    && mkdir -p $MAVEN_HOME \
    && curl -LSso /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz https://archive.apache.org/dist/maven/maven-3/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz \
    && echo "$MAVEN_DOWNLOAD_SHA512 /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz" | sha512sum -c - \
    && tar xzf /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz -C $MAVEN_HOME --strip-components=1 \
    && rm /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz \
    && update-alternatives --install /usr/bin/mvn mvn /opt/maven/bin/mvn 10000 \
    && mkdir -p $MAVEN_CONFIG_HOME \
    # Install Gradle
    && mkdir -p $GRADLE_PATH \
    && for version in $INSTALLED_GRADLE_VERSIONS; do { \
       wget -q "https://services.gradle.org/distributions/gradle-$version-all.zip" -O "$GRADLE_PATH/gradle-$version-all.zip" \
       && unzip -q "$GRADLE_PATH/gradle-$version-all.zip" -d /usr/local \
       && echo "$GRADLE_DOWNLOADS_SHA256" | grep "$version" | sed "s|$version|$GRADLE_PATH/gradle-$version-all.zip|" | sha256sum -c - \
       && rm "$GRADLE_PATH/gradle-$version-all.zip" \
       && mkdir "/tmp/gradle-$version" \
       && "/usr/local/gradle-$version/bin/gradle" -p "/tmp/gradle-$version" wrapper \
       # Android Studio uses the "-all" distribution for it's wrapper script.
       && perl -pi -e "s/gradle-$version-bin.zip/gradle-$version-all.zip/" "/tmp/gradle-$version/gradle/wrapper/gradle-wrapper.properties" \
       && "/tmp/gradle-$version/gradlew" -p "/tmp/gradle-$version" init \
       && rm -rf "/tmp/gradle-$version" \
       && if [ "$version" != "$GRADLE_VERSION" ]; then rm -rf "/usr/local/gradle-$version"; fi; \
     }; done \
    # Install default GRADLE_VERSION to path
    && ln -s /usr/local/gradle-$GRADLE_VERSION/bin/gradle /usr/bin/gradle \
    && rm -rf $GRADLE_PATH \
    # Install SBT
    && curl -fSL "https://github.com/sbt/sbt/releases/download/v${SBT_VERSION}/sbt-${SBT_VERSION}.tgz" -o sbt.tgz \
    && echo "${SBT_DOWNLOAD_SHA256} *sbt.tgz" | sha256sum -c - \
    && tar xzf sbt.tgz -C /usr/local/bin/ \
    && rm sbt.tgz \
    && for version in $LOG4J_UNSAFE_VERSIONS; do find / -name log4j*-$version.jar | xargs rm -f; done

ENV PATH "/usr/local/bin/sbt/bin:$PATH"
RUN sbt version -Dsbt.rootdir=true
# Cleanup
RUN rm -fr /var/lib/apt/lists/* /tmp/* /var/tmp/* \
    && apt-get clean
#****************     END JAVA     ****************************************************

#****************        DOCKER    *********************************************
ENV DOCKER_BUCKET="download.docker.com" \
    DOCKER_CHANNEL="stable" \
    DIND_COMMIT="3b5fac462d21ca164b3778647420016315289034" \
    DOCKER_COMPOSE_VERSION="1.26.2" \
    SRC_DIR="/usr/src"

ENV DOCKER_SHA256="945C3A3DDCB79EE7307496C2F39EB3D8372466E8654E63D60BBB462E4A3C1427"
ENV DOCKER_VERSION="20.10.22"

# Install Docker
RUN set -ex \
    && curl -fSL "https://${DOCKER_BUCKET}/linux/static/${DOCKER_CHANNEL}/x86_64/docker-${DOCKER_VERSION}.tgz" -o docker.tgz \
    && echo "${DOCKER_SHA256} *docker.tgz" | sha256sum -c - \
    && tar --extract --file docker.tgz --strip-components 1  --directory /usr/local/bin/ \
    && rm docker.tgz \
    && docker -v \
    # set up subuid/subgid so that "--userns-remap=default" works out-of-the-box
    && addgroup dockremap \
    && useradd -g dockremap dockremap \
    && echo 'dockremap:165536:65536' >> /etc/subuid \
    && echo 'dockremap:165536:65536' >> /etc/subgid \
    && wget -q "https://raw.githubusercontent.com/docker/docker/${DIND_COMMIT}/hack/dind" -O /usr/local/bin/dind \
    && curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-Linux-x86_64 > /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/dind /usr/local/bin/docker-compose \
    # Ensure docker-compose works
    && docker-compose version

VOLUME /var/lib/docker
#*********************** END  DOCKER  ****************************

#=======================End of layer: corretto  =================
FROM runtimes_n_corretto AS std_v4

# GoLang 14
ENV GOLANG_14_VERSION="1.14.15"
RUN goenv install $GOLANG_14_VERSION && rm -rf /tmp/* && goenv global  $GOLANG_14_VERSION

# Activate runtime versions specific to image version.
RUN n $NODE_12_VERSION
RUN pyenv  global $PYTHON_38_VERSION
RUN phpenv global $PHP_74_VERSION
RUN rbenv  global $RUBY_27_VERSION

# Configure SSH
COPY ssh_config /root/.ssh/config
COPY runtimes.yml /codebuild/image/config/runtimes.yml
COPY dockerd-entrypoint.sh /usr/local/bin/dockerd-entrypoint.sh
COPY legal/THIRD_PARTY_LICENSES.txt /usr/share/doc/THIRD_PARTY_LICENSES.txt
COPY legal/bill_of_material.txt     /usr/share/doc/bill_of_material.txt
COPY amazon-ssm-agent.json          /etc/amazon/ssm/amazon-ssm-agent.json

ENTRYPOINT ["/usr/local/bin/dockerd-entrypoint.sh"]

#=======================END of STD:4.0  =================
