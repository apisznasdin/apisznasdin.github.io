---
layout: post
title:  "Configuring Apache 2 HTTP Server on Solaris 10"
description: How do you configure Apache 2 HTTP Server to run automatically on Solaris 10 using rc init scripts.
permalink: /configuring-apache-2-http-server-on-solaris-10
---
How do you configure Apache 2 HTTP Server to run automatically on Solaris 10 using rc init scripts.

Log into the Solaris box as root, or su root.
Go to the /etc/init.d directory.
<code># cd /etc/init.d</code>
Copy the apache startup script to apache2
<code># cp apache apache2</code>

Change the group to sys
<code># chgrp sys apache2</code>
Next change the flags on the file to execute
<code># chmod u+x apache2</code>
Open the apache2 file with your favorite editor and modify the following lines:

<!-- <script src="https://gist.github.com/apisznasdin/5525f3c72a711d0c801a.js"></script> -->
{% highlight sh %}
#!/sbin/sh
#
# Copyright 2008 Sun Microsystems, Inc.  All rights reserved.
# Use subject to license terms.
#
#ident  "@(#)apache.sh  1.6     08/04/04 SMI"

APACHE_HOME=/usr/apache
CONF_FILE=/etc/apache/httpd.conf
RUNDIR=/var/run/apache
PIDFILE=${RUNDIR}/httpd.pid
TOMCAT_CF=/var/apache/tomcat/conf/server.xml
TOMCAT55_CF=/var/apache/tomcat55/conf/server.xml

if [ ! -f ${CONF_FILE} ]; then
        exit 0
fi

if [ ! -d ${RUNDIR} ]; then
        /usr/bin/mkdir -p -m 755 ${RUNDIR}
fi

# see if we need to start/stop tomcat also

CF=`egrep '^[ \t]*include[ \t]*/etc/apache/tomcat.conf' $CONF_FILE`
if [ -n "$CF" -a -f $TOMCAT55_CF ]; then
        TOMCAT=yes55
        TC_USER=`egrep '^[ \t]*User[ \t]' $CONF_FILE | nawk '{print $2}'`
elif [ -n "$CF" -a -f $TOMCAT_CF ]; then
        TOMCAT=yes
        TC_USER=`egrep '^[ \t]*User[ \t]' $CONF_FILE | nawk '{print $2}'`
else
        TOMCAT=no
fi

case "$1" in
start|startssl|sslstart|start-SSL)
        /bin/rm -f ${PIDFILE}
        cmdtext="starting"
        if [ "x$TOMCAT" = xyes55 ]; then
                (CATALINA_HOME=${APACHE_HOME}/tomcat55; export CATALINA_HOME; \
                    CATALINA_BASE=/var/apache/tomcat55; export CATALINA_BASE; \
                    JAVA_HOME=/usr/java; export JAVA_HOME; \
                    /bin/su $TC_USER -c \
                    "$CATALINA_HOME/bin/startup.sh") \
                    >/dev/null 2>&1
        elif [ "x$TOMCAT" != xno ]; then
                (CATALINA_HOME=${APACHE_HOME}/tomcat; export CATALINA_HOME; \
                    CATALINA_BASE=/var/apache/tomcat; export CATALINA_BASE; \
                    JAVA_HOME=/usr/java; export JAVA_HOME; \
                    /bin/su $TC_USER -c \
                    "$CATALINA_HOME/bin/startup.sh") \
                    >/dev/null 2>&1
        fi
        ;;
restart)
        cmdtext="restarting"
        ;;
stop)
        cmdtext="stopping"
        if [ "x$TOMCAT" = xyes55 ]; then
                (CATALINA_HOME=${APACHE_HOME}/tomcat55; export CATALINA_HOME; \
                    CATALINA_BASE=/var/apache/tomcat55; export CATALINA_BASE; \
                    JAVA_HOME=/usr/java; export JAVA_HOME; \
                    /bin/su $TC_USER -c \
                    "$CATALINA_HOME/bin/shutdown.sh") \
                    >/dev/null 2>&1
        elif [ "x$TOMCAT" != xno ]; then
                (CATALINA_HOME=${APACHE_HOME}/tomcat; export CATALINA_HOME; \
                    CATALINA_BASE=/var/apache/tomcat; export CATALINA_BASE; \
                    JAVA_HOME=/usr/java; export JAVA_HOME; \
                    /bin/su $TC_USER -c \
                    "$CATALINA_HOME/bin/shutdown.sh") \
                    >/dev/null 2>&1
        fi
        ;;
*)
        echo "Usage: $0 {start|stop|restart}"
        exit 1
        ;;
esac

echo "httpd $cmdtext."

/bin/sh -c "${APACHE_HOME}/bin/apachectl $1" 2>&1 &
status=$?

if [ $status != 0 ]; then
        echo "exit status $status"
        exit 1
fi
exit 0
{% endhighlight %}

as follows:

<!-- <script src="https://gist.github.com/apisznasdin/966c236ed761d0578e54.js"></script> -->

{% highlight tcsh %}
#!/sbin/sh
#
#ident  "@(#)apache2.sh"

APACHE_HOME=/usr/local/apache2
CONF_FILE=${APACHE_HOME}/conf/httpd.conf
RUNDIR=/var/run/apache2
PIDFILE=${RUNDIR}/httpd.pid

if [ ! -f ${CONF_FILE} ]; then
	exit 0
fi
if [ ! -d ${RUNDIR} ]; then
	/usr/bin/mkdir -p -m 755 ${RUNDIR}
fi

case "$1" in
start|startssl|sslstart|start-SSL)
	/bin/rm -f ${PIDFILE}
	cmdtext="starting"
	;;
restart)
	cmdtext="restarting"
	;;
stop)
	cmdtext="stopping"
	;;
*)
	echo "Usage: $0 {start|stop|restart}"
	exit 1
	;;
esac

echo "httpd $cmdtext."

/bin/sh -c "${APACHE_HOME}/bin/apachectl $1" 2>&1 &
status=$?

if [ $status != 0 ]; then
	echo "exit status $status"
	exit 1
fi
exit 0
{% endhighlight %}

Next go to the /etc/init.d directory and try executing the apache2 file. 
<code># ./apache2 start</code>
You should see something like this:
<blockquote>httpd starting.</blockquote>
You can check your configuration by using a browser to check it.

Next you will want to configure it to start/stop at the various runlevels.

Go to /etc/rc0.d and add the following:
<code># cd ../rc0.d</code>
<code># ln -s ../init.d/apache2 K17apache2</code>
Go to /etc/rc1.d and add the following:
<code># cd ../rc1.d</code>
<code># ln -s ../init.d/apache2 K17apache2</code>
Go to /etc/rc3.d and add the following:
<code># cd ../rc3.d</code>
<code># ln -s ../init.d/apache2 S51apache2</code>
Go to /etc/rcS.d and add the following:
<code># cd ../rcS.d</code>
<code># ln -s ../init.d/apache2 K17apache2</code>

This will complete the configuration for your server. Next time you change run levels, or reboot, the Apache 2 HTTP Server will restart as you expect.
