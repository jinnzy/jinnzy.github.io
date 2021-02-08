# elasticsearch docker镜像添加插件


没使用docker之前，在安装/升级es时处理插件特别麻烦，每个节点都需要安装/升级对应插件，使用docker后只需要把需要用到的插件预先安装到镜像中就可以了。

## Dockerfile

在官方提供的镜像基础上利用多阶段构建添加了`ik` `pinyin` `repository-s3` 等插件

```
FROM alpine:3.9 as builder
WORKDIR /opt
RUN echo "http://mirrors.aliyun.com/alpine/v3.9/main/" > /etc/apk/repositories && \
    echo "http://mirrors.aliyun.com/alpine/v3.9/community/" >> /etc/apk/repositories && \
    apk update && \
    apk upgrade && \
    apk add wget && \
    wget -O ik.zip  https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.10.2/elasticsearch-analysis-ik-{{ES_VERSION}}.zip && \
    wget -O pinyin.zip https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v7.10.2/elasticsearch-analysis-pinyin-{{ES_VERSION}}.zip && \
    wget -O repository-s3.zip https://artifacts.elastic.co/downloads/elasticsearch-plugins/repository-s3/repository-s3-v7.10.2.zip
FROM docker.elastic.co/elasticsearch/elasticsearch:7.10.2
COPY --from=builder  /opt/ik.zip /tmp/
COPY --from=builder  /opt/pinyin.zip /tmp/
COPY --from=builder /opt/repository-s3.zip /tmp/
COPY ./bin/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
RUN bin/elasticsearch-plugin install -b file:///tmp/ik.zip &&\
    bin/elasticsearch-plugin install -b file:///tmp/pinyin.zip &&\
    bin/elasticsearch-plugin install -b file:///tmp/repository-s3.zip &&\
    rm -f /tmp/ik.zip &&\
    rm -f /tmp/pinyin.zip && \
    rm -f /tmp/repository-s3.zip && \
    chown 1000.1000 /usr/local/bin/docker-entrypoint.sh && \
    chmod 755 /usr/local/bin/docker-entrypoint.sh
```

`bin/docker-entrypoint.sh` 入口启动文件，这个文件随着版本升级会有改动，制作镜像前最好查看一下目标版本的[入口文件](https://github.com/elastic/dockerfiles/blob/7.10/elasticsearch/bin/docker-entrypoint.sh)。

在官方提供的`docker-entrypoint.sh`基础上添加了一些插件相关的配置变量。

```
#!/bin/bash
set -e

# Files created by Elasticsearch should always be group writable too
umask 0002

run_as_other_user_if_needed() {
  if [[ "$(id -u)" == "0" ]]; then
    # If running as root, drop to specified UID and run command
    exec chroot --userspec=1000 / "${@}"
  else
    # Either we are running in Openshift with random uid and are a member of the root group
    # or with a custom --user
    exec "${@}"
  fi
}

# Allow user specify custom CMD, maybe bin/elasticsearch itself
# for example to directly specify `-E` style parameters for elasticsearch on k8s
# or simply to run /bin/bash to check the image
if [[ "$1" != "eswrapper" ]]; then
  if [[ "$(id -u)" == "0" && $(basename "$1") == "elasticsearch" ]]; then
    # centos:7 chroot doesn't have the `--skip-chdir` option and
    # changes our CWD.
    # Rewrite CMD args to replace $1 with `elasticsearch` explicitly,
    # so that we are backwards compatible with the docs
    # from the previous Elasticsearch versions<6
    # and configuration option D:
    # https://www.elastic.co/guide/en/elasticsearch/reference/5.6/docker.html#_d_override_the_image_8217_s_default_ulink_url_https_docs_docker_com_engine_reference_run_cmd_default_command_or_options_cmd_ulink
    # Without this, user could specify `elasticsearch -E x.y=z` but
    # `bin/elasticsearch -E x.y=z` would not work.
    set -- "elasticsearch" "${@:2}"
    # Use chroot to switch to UID 1000
    exec chroot --userspec=1000 / "$@"
  else
    # User probably wants to run something else, like /bin/bash, with another uid forced (Openshift?)
    exec "$@"
  fi
fi

# Allow environment variables to be set by creating a file with the
# contents, and setting an environment variable with the suffix _FILE to
# point to it. This can be used to provide secrets to a container, without
# the values being specified explicitly when running the container.
#
# This is also sourced in elasticsearch-env, and is only needed here
# as well because we use ELASTIC_PASSWORD below. Sourcing this script
# is idempotent.
source /usr/share/elasticsearch/bin/elasticsearch-env-from-file

if [[ -f bin/elasticsearch-users ]]; then
  # Check for the ELASTIC_PASSWORD environment variable to set the
  # bootstrap password for Security.
  #
  # This is only required for the first node in a cluster with Security
  # enabled, but we have no way of knowing which node we are yet. We'll just
  # honor the variable if it's present.
  if [[ -n "$ELASTIC_PASSWORD" ]]; then
    [[ -f /usr/share/elasticsearch/config/elasticsearch.keystore ]] || (run_as_other_user_if_needed elasticsearch-keystore create)
    if ! (run_as_other_user_if_needed elasticsearch-keystore has-passwd --silent) ; then
      # keystore is unencrypted
      if ! (run_as_other_user_if_needed elasticsearch-keystore list | grep -q '^bootstrap.password$'); then
        (run_as_other_user_if_needed echo "$ELASTIC_PASSWORD" | elasticsearch-keystore add -x 'bootstrap.password')
      fi
    else
      # keystore requires password
      if ! (run_as_other_user_if_needed echo "$KEYSTORE_PASSWORD" \
          | elasticsearch-keystore list | grep -q '^bootstrap.password$') ; then
        COMMANDS="$(printf "%s\n%s" "$KEYSTORE_PASSWORD" "$ELASTIC_PASSWORD")"
        (run_as_other_user_if_needed echo "$COMMANDS" | elasticsearch-keystore add -x 'bootstrap.password')
      fi
    fi
  fi
fi

if [[ "$(id -u)" == "0" ]]; then
  # If requested and running as root, mutate the ownership of bind-mounts
  if [[ -n "$TAKE_FILE_OWNERSHIP" ]]; then
    chown -R 1000:0 /usr/share/elasticsearch/{data,logs}
  fi
fi
# ik相关配置
if [ -n "$REMOTE_EXT_DICT" ]; then
sed -i 's#<!-- <entry key="remote_ext_dict">words_location</entry> -->#<entry key="remote_ext_dict">'''${REMOTE_EXT_DICT}'''</entry>#g'  /usr/share/elasticsearch/config/analysis-ik/IKAnalyzer.cfg.xml
fi
if [ -n "$REMOTE_EXT_STOPWORDS" ]; then
sed -i 's#<!-- <entry key="remote_ext_stopwords">words_location</entry> -->#<entry key="remote_ext_stopwords">'''${REMOTE_EXT_STOPWORDS}'''</entry>#g'  /usr/share/elasticsearch/config/analysis-ik/IKAnalyzer.cfg.xml
fi
# s3配置
echo "-Des.allow_insecure_settings=true" >> /usr/share/elasticsearch/config/jvm.options

run_as_other_user_if_needed /usr/share/elasticsearch/bin/elasticsearch <<<"$KEYSTORE_PASSWORD"
```

## 构建镜像

```
docker build -t  elasticsearch:7.10.2-customize .
```

这样后面安装或者升级都不需要操心插件之类的问题了，直接替换镜像即可。

**备注**：上面这些文件已经放到[github](https://github.com/jinnzy/config-repo/tree/master/docker-file/elasticsearch)上了，如果有需要也可以从这里拉取

