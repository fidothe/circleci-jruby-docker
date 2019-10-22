FROM openjdk:8-jdk-slim

RUN apt-get update && apt-get install -y libc6-dev curl --no-install-recommends && rm -rf /var/lib/apt/lists/*

ENV JRUBY_SHA 958d406cd7e908d55a489cf682df1dc1bfbd13ee
RUN mkdir /opt/jruby \
  && curl -fSL https://oss.sonatype.org/content/repositories/snapshots/org/jruby/jruby-dist/9.2.9.0-SNAPSHOT/jruby-dist-9.2.9.0-20191021.165226-75-bin.tar.gz -o /tmp/jruby.tar.gz \
  && echo "$JRUBY_SHA /tmp/jruby.tar.gz" | sha1sum -c - \
  && tar -zx --strip-components=1 -f /tmp/jruby.tar.gz -C /opt/jruby \
  && rm /tmp/jruby.tar.gz \
  && update-alternatives --install /usr/local/bin/ruby ruby /opt/jruby/bin/jruby 1
ENV PATH /opt/jruby/bin:$PATH

# skip installing gem documentation
RUN mkdir -p /opt/jruby/etc \
	&& { \
		echo 'install: --no-document'; \
		echo 'update: --no-document'; \
	} >> /opt/jruby/etc/gemrc

RUN gem install bundler rake net-telnet xmlrpc

# install things globally, for great justice
# and don't create ".bundle" in all our apps
ENV GEM_HOME /usr/local/bundle
ENV BUNDLE_PATH="$GEM_HOME" \
	BUNDLE_BIN="$GEM_HOME/bin" \
	BUNDLE_SILENCE_ROOT_WARNING=1 \
	BUNDLE_APP_CONFIG="$GEM_HOME"
ENV PATH $BUNDLE_BIN:$PATH
RUN mkdir -p "$GEM_HOME" "$BUNDLE_BIN" \
	&& chmod 777 "$GEM_HOME" "$BUNDLE_BIN"

CMD [ "irb" ]
