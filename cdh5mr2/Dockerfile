FROM ubuntu:12.04

MAINTAINER "Ryan Hafen" rhafen@gmail.com

# fix bash bug
RUN apt-get update -q -q && apt-get dist-upgrade --yes --force-yes

# install sudo
RUN apt-get update -q -q
RUN apt-get install -y sudo vim wget curl

RUN su -c "useradd tessera -s /bin/bash -m"
RUN chpasswd << 'END' \
tessera:dnr \
END

cd /home/tessera

## Add Cloudera repositories
RUN wget -c http://archive.cloudera.com/cdh5/one-click-install/precise/amd64/cdh5-repository_1.0_all.deb
RUN dpkg -i cdh5-repository_1.0_all.deb
RUN curl -s http://archive.cloudera.com/cdh5/ubuntu/precise/amd64/cdh/archive.key | apt-key add -

RUN apt-get update -q -q

## Install Java
RUN apt-get install -y openjdk-7-jdk

## Install Hadoop with YARN
RUN apt-get install -y hadoop-conf-pseudo
RUN dpkg -L hadoop-conf-pseudo

## Format the NameNode
RUN sudo -u hdfs hdfs namenode -format

## Start HDFS
RUN for x in `cd /etc/init.d ; ls hadoop-hdfs-*` ; do service $x start ; done

## Create the new directories and set permissions:
RUN sudo -u hdfs hadoop fs -mkdir -p /tmp/hadoop-yarn/staging/history/done_intermediate
RUN sudo -u hdfs hadoop fs -chown -R mapred:mapred /tmp/hadoop-yarn/staging
RUN sudo -u hdfs hadoop fs -chmod -R 1777 /tmp
RUN sudo -u hdfs hadoop fs -mkdir -p /var/log/hadoop-yarn
RUN sudo -u hdfs hadoop fs -chown yarn:mapred /var/log/hadoop-yarn

## Verify the HDFS File Structure
# sudo -u hdfs hadoop fs -ls -R /

## Start YARN
RUN service hadoop-yarn-resourcemanager start
RUN service hadoop-yarn-nodemanager start
RUN service hadoop-mapreduce-historyserver start

## Create User Directories
RUN sudo -u hdfs hadoop fs -mkdir /user
RUN sudo -u hdfs hadoop fs -chown tessera /user
RUN sudo -u hdfs hadoop fs -mkdir /user/tessera
RUN sudo -u hdfs hadoop fs -chown tessera /user/tessera

## Install HiveServer2
RUN sudo apt-get install -y hive-server2

RUN rm cdh5-repository*

###########

RUN function eVal { \
    echo $1 | tee -a /home/tessera/.Renviron \
    # echo $1 | tee -a /etc/R/Renviron \
    echo export $1 | tee -a /home/tessera/.bashrc \
    # echo export $1 | tee -a /etc/profile \
}

RUN eVal 'HADOOP=/usr/lib/hadoop'
RUN eVal 'HADOOP_HOME=/usr/lib/hadoop'
RUN eVal 'HADOOP_CONF_DIR=/etc/hadoop/conf'
RUN eVal 'HADOOP_BIN=$HADOOP_HOME/bin'
RUN eVal 'HADOOP_OPTS=-Djava.awt.headless=true'
RUN eVal 'HADOOP_LIBS=/etc/hadoop/conf:/usr/lib/hadoop/lib/:/usr/lib/hadoop/.//:/usr/lib/hadoop-hdfs/./:/usr/lib/hadoop-hdfs/lib/:/usr/lib/hadoop-hdfs/.//:/usr/lib/hadoop-yarn/lib/:/usr/lib/hadoop-yarn/.//:/usr/lib/hadoop-mapreduce/lib/:/usr/lib/hadoop-mapreduce/.//'
RUN eVal 'LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH'
RUN eVal 'RHIPE_RUNNER=/home/tessera/rhRunner.sh'
RUN eVal 'JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64'

RUN chown -R tessera:tessera .

RUN echo '/usr/lib/jvm/java-7-openjdk-amd64/jre/lib/amd64/server/' | tee -a  /etc/ld.so.conf.d/jre.conf
RUN echo '/usr/lib/jvm/java-7-openjdk-amd64/jre/lib/amd64/' | tee -a  /etc/ld.so.conf.d/jre.conf
RUN echo '/usr/lib/hadoop/lib' | tee -a  /etc/ld.so.conf.d/hadoop.conf

## build/install R

RUN apt-get -y update
RUN apt-get -y install pkg-config unzip libcairo2-dev libcurl4-openssl-dev screen libssl0.9.8 gdebi-core firefox libapparmor1 psmisc supervisor

RUN echo "deb http://cran.rstudio.com/bin/linux/ubuntu precise/" | tee -a /etc/apt/sources.list
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys E084DAB9

RUN apt-get -y update

#### install R
RUN apt-get install -y r-base-dev
RUN chmod -R aou=rwx /usr/local/lib/R/site-library

RUN R CMD javareconf

USER tessera

## rJava package
RUN R -e "install.packages('rJava', repos='http://www.rforge.net/')"

## shiny package
RUN R -e "install.packages('shiny', repos='http://cran.rstudio.com/')"

USER root

## rstudio

RUN update-locale

RUN wget -q https://s3.amazonaws.com/rstudio-server/current.ver -O currentVersion.txt
RUN ver=$(cat currentVersion.txt)
RUN wget http://download2.rstudio.org/rstudio-server-${ver}-amd64.deb
RUN dpkg -i rstudio-server-${ver}-amd64.deb
RUN rm rstudio-server-*-amd64.deb currentVersion.txt

# put rstudio on part 80
RUN echo "www-port=80" | tee -a /etc/rstudio/rserver.conf
RUN echo "rsession-ld-library-path=/usr/local/lib" | tee -a /etc/rstudio/rserver.conf
RUN rstudio-server restart

## shiny server
RUN ver=$(wget -qO- https://s3.amazonaws.com/rstudio-shiny-server-os-build/ubuntu-12.04/x86_64/VERSION)
RUN wget https://s3.amazonaws.com/rstudio-shiny-server-os-build/ubuntu-12.04/x86_64/shiny-server-${ver}-amd64.deb -O shiny-server.deb
RUN dpkg -i shiny-server.deb
RUN rm shiny-server.deb

## copy shiny examples
RUN mkdir /srv/shiny-server/examples
RUN cp -R /usr/local/lib/R/site-library/shiny/examples/* /srv/shiny-server/examples
RUN chown -R shiny:shiny /srv/shiny-server/examples

## allow tessera user to sudo chown shiny-server directory
RUN echo "tessera ALL=(ALL) NOPASSWD: /bin/chown -R shiny /srv/shiny-server" | tee -a /etc/sudoers

## protobuf
RUN export PROTO_BUF_VERSION=2.5.0
RUN wget https://protobuf.googlecode.com/files/protobuf-$PROTO_BUF_VERSION.tar.bz2
RUN tar jxvf protobuf-$PROTO_BUF_VERSION.tar.bz2
RUN cd protobuf-$PROTO_BUF_VERSION
RUN ./configure && make -j4
RUN make install
RUN cd ..
RUN rm -rf protobuf-*

## RHIPE
USER tessera

RUN export RHIPE_VERSION=Rhipe_0.75.0_cdh5mr2
RUN export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH
RUN export PKG_CONFIG_PATH=/usr/local/lib
RUN wget http://ml.stat.purdue.edu/rhipebin/$RHIPE_VERSION.tar.gz

RUN R CMD INSTALL $RHIPE_VERSION.tar.gz
RUN rm $RHIPE_VERSION.tar.gz

## RHIPE runner
RUN echo "export LD_LIBRARY_PATH=/usr/local/lib" | tee -a /home/tessera/rhRunner.sh
RUN echo "exec /usr/bin/R CMD /usr/local/lib/R/site-library/Rhipe/bin/RhipeMapReduce --slave --silent --vanilla" | tee -a /home/tessera/rhRunner.sh

RUN chown -R tessera:tessera /home/tessera
RUN chmod 755 /home/tessera
RUN chmod 755 /home/tessera/rhRunner.sh

USER root

## components needed to build RHIPE from source
RUN apt-get -y install git ant maven

USER tessera

## more packages
R -e "install.packages('devtools', repos='http://cran.rstudio.com/')"
R -e "options(unzip = 'unzip', repos = 'http://cran.rstudio.com/'); library(devtools); install_github('tesseradata/datadr')"
R -e "options(unzip = 'unzip', repos = 'http://cran.rstudio.com/'); library(devtools); install_github('tesseradata/trelliscope')"
R -e "install.packages('testthat', repos='http://cran.rstudio.com/')"
R -e "install.packages('roxygen2', repos='http://cran.rstudio.com/')"

USER root

## Configure persistent daemons
RUN mkdir -p /var/log/supervisor
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

EXPOSE 22 80 8787 8030 8031 8032 8033 8040 8042 8088 9000 50010 50020 50070 50075 50090

## Run persistent tasks (see supervisord.conf)
CMD ["/usr/bin/supervisord"] 

USER tessera

