FROM alpine AS deploy

# Install build dependencies
RUN apk add --no-cache afl++ autoconf automake chromium clang clang-analyzer clang-extra-tools cmake curl doxygen file fontconfig gdb g++ git git-lfs libtool linux-headers make msttcorefonts-installer nodejs patch perl py3-colorlog py3-jinja2 py3-lxml py3-setuptools python3 samurai sed util-linux-dev valgrind xz && \
	ln -sf python3 /usr/bin/python

# Install npm, JDK, gradle from latest-stable for security fixes
RUN apk add --no-cache -X https://dl-cdn.alpinelinux.org/alpine/latest-stable/community npm openjdk17-jdk && \
	curl -sL $(curl -sL https://gradle.org/releases/ | sed -nre 's/.*"([^"]+-bin\.zip)".*/\1/p' | head -n1) | unzip -d /opt -q - && \
	chmod +x /opt/gradle-*/bin/gradle && ln -s /opt/gradle-*/bin/gradle /usr/bin/gradle

# Install Powershell and its dependencies, see https://learn.microsoft.com/en-us/powershell/scripting/install/install-alpine
RUN apk add --no-cache ca-certificates less ncurses-terminfo-base krb5-libs libgcc libintl libssl3 libstdc++ tzdata userspace-rcu zlib icu-libs curl lttng-ust && \
	mkdir -p /opt/microsoft/pwsh && \
	curl -sL $(curl -sL https://api.github.com/repos/PowerShell/PowerShell/releases/latest | sed -nre 's/"browser_download_url"\s*:\s*"([^"]+-musl-x(86_)?64\.tar\.gz)"/\1/p') | tar zx -C /opt/microsoft/pwsh && \
	chmod +x /opt/microsoft/pwsh/pwsh && \
	ln -s /opt/microsoft/pwsh/pwsh /usr/bin/pwsh

# Install erlang
RUN apk add --no-cache erlang

# Install cross compilers
RUN for Arch in aarch64-linux-musl arm-linux-musleabihf i686-linux-musl powerpc64le-linux-musl powerpc-linux-musl x86_64-linux-musl x86_64-linux-muslx32; do \
	mkdir -p "/opt/$Arch" && \
	curl -sL "https://github.com/76-eddge/musl-cross/releases/download/v0.9.43/$Arch.tar.xz" | tar xJ -C "/opt/$Arch"; \
	done; \
	find /opt/*/*/include/c++ -name c++config.h | xargs sed -i -re 's/^(\s*#\s*define\s+_GLIBCXX_HAVE_ATTRIBUTE_VISIBILITY).*/\1 0/'

# Perform tasks as basic user
RUN adduser --disabled-password jenkins
USER jenkins
WORKDIR /home/jenkins

# Install fonts for user
RUN mkdir -p ~/.fonts && \
	curl -s -L https://github.com/mrbvrz/segoe-ui-linux/archive/02f6e1e9290581e7fc9bf3beefc0ebc9ec8c1611.zip | unzip -d ~/.fonts -j - '*/font/*' && \
	curl -s -L -o ~/.fonts/seguisym.ttf https://github.com/microsoft/elfie-arriba/raw/6889a336985caf20ac596a7ff681362423ef5052/Arriba/Arriba.Web/fonts/SEGUISYM.TTF && \
	curl -s -L http://plasmasturm.org/dl/vistafonts-installer | sed -e 's/{,b,i,z}/*/g' | sh && \
	rm -f *.exe

# Initialize npm
COPY --chown=jenkins package.json ./npm-init/
RUN PUPPETEER_SKIP_DOWNLOAD=true cd npm-init && \
	npm install && \
	cd .. && \
	rm -rf npm-init

# Initialize git
RUN git config --global --add safe.directory "*"

# Initialize gradle
COPY --chown=jenkins build.gradle ./gradle-init/
RUN cd gradle-init && \
	gradle -i --no-daemon resolveDependencies --refresh-dependencies && \
	cd .. && \
	rm -rf gradle-init

# Initialize PowerShell
SHELL ["/usr/bin/pwsh", "-Command"]
RUN New-Item -ItemType File -Path $Profile -Force; \
	Add-Content $Profile '$env:PUPPETEER_EXECUTABLE_PATH = "/usr/bin/chromium-browser"'
