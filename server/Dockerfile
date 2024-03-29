# This Dockerflle can be used to build and configure an image of the Payara Application Server
# for hosting SOLA solutions. Note that pre-built docker images for the SOLA solutions are
# available from DockerHub at https://hub.docker.com/u/mcdowella

# Avoid versions 5.182, 5.183, and 5.184 of Payara server. These versions have been
# tested, but prevent SOLA from deploying/running due to various issues.
FROM payara/server-full:5.181

# Uncomment PAYARA_PATH and use for versions from 5.183 and above. PAYARA_PATH is set in
# versions prior to 5.183 by the Payara build.
#ENV PAYARA_PATH $PAYARA_DIR

# SOLA solution and version to build.
# Docker arguments can be overridden from the build command line. See the --build-arg option.
ARG SOLA_VERSION=1503a

# SOLA Registry settings
#ARG SOLA_SOLUTION=registry
#ARG CONTEXT_ROOT=sola
#ARG SOLA_SERVICES_EAR=sola-services-ear-${SOLA_VERSION}
#ARG SOLA_DESKTOP_WEBSTART=desktop-web-start-${SOLA_VERSION}
#ARG SOLA_WEB_ADMIN=sola-web-admin-1504a

# SOLA State Land Settings
ARG SOLA_SOLUTION=stateland
ARG CONTEXT_ROOT=sola_sl
ARG SOLA_SERVICES_EAR=sola-sl-services-ear-${SOLA_VERSION}
ARG SOLA_DESKTOP_WEBSTART=desktop-sl-web-start-${SOLA_VERSION}
ARG SOLA_WEB_ADMIN=sola-web-admin-1504a

#SOLA Systematic Registration Settings
#ARG SOLA_SOLUTION=systemreg
#ARG CONTEXT_ROOT=sola_sr
#ARG SOLA_SERVICES_EAR=sola-sr-services-ear-${SOLA_VERSION}
#ARG SOLA_DESKTOP_WEBSTART=desktop-sr-web-start-${SOLA_VERSION}
#ARG SOLA_WEB_ADMIN=sola-web-admin-1504a


# Keystore parameters
ARG SRC_KEYSTORE_PASS=changeit
ARG MSG_ENCRYPT_CERT_ALIAS=xws-security-server

# Database connection parameters
ARG DATABASE_USER=postgres
ARG DATABASE_PASS=sola
ARG DATABASE=sola
ARG DATAPOOL=sola
ENV DATABASE_HOST=database
ENV DATABASE_PORT=5432

# Payara domain and deploy parameters
ARG PAYARA_DOMAIN=domain1
ARG PAYARA_PASSWD=admin

# The max length of time for the server to wait before starting the Payara domain.
# Can be used to give the SOLA Database server time to initialize.
ENV WAIT_TIME=90

RUN mkdir -p ${PAYARA_PATH}/temp/ && \
    mkdir -p ${PAYARA_PATH}/sola_deploy/

COPY common/ ${SOLA_SOLUTION}/certs/ ${PAYARA_PATH}/temp/
# Copy EAR and WAR packages to deploy based on the selected SOLA version.
COPY --chown=payara:payara ${SOLA_SOLUTION}/deploy/${SOLA_VERSION}/ ${PAYARA_PATH}/sola_deploy/

# Add the certificate used by SOLA for message encryption to the Payara keystore.
# IMPORTANT: This certificate is for demonstration purposes only. For production deployment,
# create a new certificate and ensure the private key is kept confidential.
RUN keytool -importkeystore -srckeystore ${PAYARA_PATH}/temp/keystore.jks -destkeystore ${PAYARA_PATH}/glassfish/domains/${PAYARA_DOMAIN}/config/keystore.jks -srcstorepass ${SRC_KEYSTORE_PASS} -deststorepass changeit -srcalias ${MSG_ENCRYPT_CERT_ALIAS} -noprompt && \
    keytool -importkeystore -srckeystore ${PAYARA_PATH}/temp/cacerts.jks -destkeystore ${PAYARA_PATH}/glassfish/domains/${PAYARA_DOMAIN}/config/cacerts.jks -srcstorepass ${SRC_KEYSTORE_PASS} -deststorepass changeit -srcalias ${MSG_ENCRYPT_CERT_ALIAS} -noprompt

USER root

# Copy files required for SOLA into the domain and sets up the file access permissions
RUN mv ${PAYARA_PATH}/temp/jackson* ${PAYARA_PATH}/glassfish/modules/. && \
	mv ${PAYARA_PATH}/temp/postgresql-42* ${PAYARA_PATH}/glassfish/domains/${PAYARA_DOMAIN}/lib/ && \
	mv ${PAYARA_PATH}/temp/init_2_delay_start.sh ${PAYARA_PATH}/. && \
	mv ${PAYARA_PATH}/temp/wait_for_it.sh ${PAYARA_PATH}/. && \
	chown -R payara:payara ${PAYARA_PATH}/glassfish/modules/jackson* && \
    chmod 755 ${PAYARA_PATH}/glassfish/modules/jackson* && \
    chown -R payara:payara ${PAYARA_PATH}/glassfish/domains/${PAYARA_DOMAIN}/lib/*.jar && \
	chmod 755 ${PAYARA_PATH}/glassfish/domains/${PAYARA_DOMAIN}/lib/*.jar && \
	chown -R payara:payara ${PAYARA_PATH}/glassfish/domains/${PAYARA_DOMAIN}/config/ && \
	chmod 755 ${PAYARA_PATH}/init_2_delay_start.sh && \
	chown -R payara:payara ${PAYARA_PATH}/init_2_delay_start.sh && \
	chmod 755 ${PAYARA_PATH}/wait_for_it.sh && \
	chown -R payara:payara ${PAYARA_PATH}/wait_for_it.sh

USER payara

# RUN completes the following tasks:
# 1. Creates a command file to configure the SOLA domain to support deployment of the SOLA applications.
# 2. Creates password file to access Payara. Assumes default password has been set to admin.
#    Check the payara/server-full image for password changes.
# 3. Creates a custom post boot deploy commands that will deploy the SOLA applications (sola-services,
#    desktop-web-start and the sola-web-admin)
#    the first time the Payara image is run. Uses POSTBOOT_COMMANDS environment variable setup by
#    the payara/server-full image.
# 4. Configures the domain for SOLA deployment using the admin command file (created in step 1) -
#    but does not deploy the applications. Deploying the apps is performed when the image is
#    run to create a container.
RUN echo "create-jdbc-connection-pool --validateAtmostOncePeriodInSeconds=0 --sqlTraceListeners=fish.payara.jdbc.SilentSqlTraceListener --statementLeakReclaim=false --ping=false --maxWaitTimeInMillis=60000 --wrapJdbcObjects=true --connectionValidationMethod=table --steadyPoolSize=8 --idleTimeoutInSeconds=300 --maxConnectionUsageCount=0 --matchConnections=false --connectionCreationRetryIntervalInSeconds=10 --associateWithThread=false --isConnectionValidationRequired=false --connectionLeakReclaim=false --datasourceClassname=org.postgresql.ds.PGConnectionPoolDataSource --connectionCreationRetryAttempts=0 --statementCacheSize=0 --resType=javax.sql.ConnectionPoolDataSource --statementLeakTimeoutInSeconds=0 --failAllConnections=false --statementTimeoutInSeconds=-1 --lazyConnectionEnlistment=false --pooling=true --poolResizeQuantity=2 --allowNonComponentCallers=false --connectionLeakTimeoutInSeconds=0 --isIsolationLevelGuaranteed=true --maxPoolSize=32 --nonTransactionalConnections=false --lazyConnectionAssociation=false ${DATAPOOL} \n\
set resources.jdbc-connection-pool.${DATAPOOL}.property.TargetServerType=any resources.jdbc-connection-pool.${DATAPOOL}.property.BinaryTransfer=true resources.jdbc-connection-pool.${DATAPOOL}.property.UnknownLength=2147483647 resources.jdbc-connection-pool.${DATAPOOL}.property.DisableColumnSanitiser=false resources.jdbc-connection-pool.${DATAPOOL}.property.UseSpNego=false resources.jdbc-connection-pool.${DATAPOOL}.property.SspiServiceClass=POSTGRES resources.jdbc-connection-pool.${DATAPOOL}.property.ProtocolVersion=0 resources.jdbc-connection-pool.${DATAPOOL}.property.SendBufferSize=-1 resources.jdbc-connection-pool.${DATAPOOL}.property.ReceiveBufferSize=-1 resources.jdbc-connection-pool.${DATAPOOL}.property.LoadBalanceHosts=false resources.jdbc-connection-pool.${DATAPOOL}.property.ReadOnly=false resources.jdbc-connection-pool.${DATAPOOL}.property.LogUnclosedConnections=false resources.jdbc-connection-pool.${DATAPOOL}.property.DatabaseName=${DATABASE} resources.jdbc-connection-pool.${DATAPOOL}.property.GssLib=auto resources.jdbc-connection-pool.${DATAPOOL}.property.CancelSignalTimeout=10 resources.jdbc-connection-pool.${DATAPOOL}.property.PortNumber=${DATABASE_PORT} resources.jdbc-connection-pool.${DATAPOOL}.property.DefaultRowFetchSize=0 resources.jdbc-connection-pool.${DATAPOOL}.property.User=${DATABASE_USER} resources.jdbc-connection-pool.${DATAPOOL}.property.DatabaseMetadataCacheFieldsMiB=5 resources.jdbc-connection-pool.${DATAPOOL}.property.URL=jdbc:postgresql://${DATABASE_HOST}/${DATABASE} resources.jdbc-connection-pool.${DATAPOOL}.property.Url=jdbc:postgresql://${DATABASE_HOST}/${DATABASE} resources.jdbc-connection-pool.${DATAPOOL}.property.AllowEncodingChanges=false resources.jdbc-connection-pool.${DATAPOOL}.property.JaasLogin=true resources.jdbc-connection-pool.${DATAPOOL}.property.PrepareThreshold=5 resources.jdbc-connection-pool.${DATAPOOL}.property.ReWriteBatchedInserts=false resources.jdbc-connection-pool.${DATAPOOL}.property.SocketTimeout=0 resources.jdbc-connection-pool.${DATAPOOL}.property.HostRecheckSeconds=10 resources.jdbc-connection-pool.${DATAPOOL}.property.Password=${DATABASE_PASS} resources.jdbc-connection-pool.${DATAPOOL}.property.ConnectTimeout=10 resources.jdbc-connection-pool.${DATAPOOL}.property.ApplicationName='PostgreSQL JDBC Driver' resources.jdbc-connection-pool.${DATAPOOL}.property.PreparedStatementCacheQueries=256 resources.jdbc-connection-pool.${DATAPOOL}.property.Ssl=false resources.jdbc-connection-pool.${DATAPOOL}.property.PreparedStatementCacheSizeMiB=5 resources.jdbc-connection-pool.${DATAPOOL}.property.LoginTimeout=0 resources.jdbc-connection-pool.${DATAPOOL}.property.ServerName=${DATABASE_HOST} resources.jdbc-connection-pool.${DATAPOOL}.property.DatabaseMetadataCacheFields=65536 resources.jdbc-connection-pool.${DATAPOOL}.property.TcpKeepAlive=false \n\
create-jdbc-resource --enabled=true --poolName=${DATAPOOL} --target=domain jdbc/${DATAPOOL} \n\
create-resource-ref --enabled=true --target=server jdbc/${DATAPOOL} \n\
create-jdbc-resource --enabled=true --poolName=${DATAPOOL} --target=domain jdbc/${DATAPOOL}_sl \n\
create-resource-ref --enabled=true --target=server jdbc/${DATAPOOL}_sl \n\
create-jdbc-resource --enabled=true --poolName=${DATAPOOL} --target=domain jdbc/${DATAPOOL}_sr \n\
create-resource-ref --enabled=true --target=server jdbc/${DATAPOOL}_sr \n\
create-auth-realm --classname=com.sun.enterprise.security.auth.realm.jdbc.JDBCRealm --property=jaas-context=jdbcRealm:datasource-jndi=jdbc/${DATAPOOL}:user-table=system.active_users:user-name-column=username:password-column=passwd:group-table=system.user_roles:group-name-column=rolename: --target=server-config SolaRealm \n\
create-auth-realm --classname=com.sun.enterprise.security.auth.realm.jdbc.JDBCRealm --property=jaas-context=jdbcRealm:datasource-jndi=jdbc/${DATAPOOL}_sl:user-table=system.active_users:user-name-column=username:password-column=passwd:group-table=system.user_roles:group-name-column=rolename: --target=server-config SolaSLRealm \n\
create-auth-realm --classname=com.sun.enterprise.security.auth.realm.jdbc.JDBCRealm --property=jaas-context=jdbcRealm:datasource-jndi=jdbc/${DATAPOOL}_sr:user-table=system.active_users:user-name-column=username:password-column=passwd:group-table=system.user_roles:group-name-column=rolename: --target=server-config SolaSRRealm \n\
set-log-levels --target=server-config org.sola.services=INFO:java.sql.ResultSet=OFF:java.sql.Connection=FINE:java.sql=FINE " > ${PAYARA_PATH}/temp/asadmin-command-file && \
    echo "AS_ADMIN_PASSWORD=${PAYARA_PASSWD}" > ${PAYARA_PATH}/temp/passwordFile && \
    echo "deploy --keepState=false --precompilejsp=false --availabilityEnabled=false --name=${SOLA_SERVICES_EAR} --verify=false --force=false --enabled=true --properties=implicitCdiEnabled=true:preserveAppScopedResources=false:java-web-start-enabled=true --target=server ${PAYARA_PATH}/sola_deploy/${SOLA_SERVICES_EAR}.ear \n\
deploy --keepState=false --precompilejsp=false --availabilityEnabled=false --name=${SOLA_WEB_ADMIN} --verify=false --force=false --contextroot=${CONTEXT_ROOT}/admin --enabled=true --properties=implicitCdiEnabled=true:preserveAppScopedResources=false:cdiDevModeEnabled=false --target=server ${PAYARA_PATH}/sola_deploy/${SOLA_WEB_ADMIN}.war \n\
deploy --keepState=false --precompilejsp=false --availabilityEnabled=false --name=${SOLA_DESKTOP_WEBSTART} --verify=false --force=false --contextroot=${CONTEXT_ROOT} --enabled=true --properties=implicitCdiEnabled=true:preserveAppScopedResources=false:cdiDevModeEnabled=false --target=server ${PAYARA_PATH}/sola_deploy/${SOLA_DESKTOP_WEBSTART}.war " > ${POSTBOOT_COMMANDS} && \
    ${PAYARA_PATH}/bin/asadmin -u admin -W ${PAYARA_PATH}/temp/passwordFile start-domain ${PAYARA_DOMAIN} && \
    ${PAYARA_PATH}/bin/asadmin -u admin -W ${PAYARA_PATH}/temp/passwordFile multimode --file ${PAYARA_PATH}/temp/asadmin-command-file && \
    ${PAYARA_PATH}/bin/asadmin -u admin -W ${PAYARA_PATH}/temp/passwordFile stop-domain ${PAYARA_DOMAIN}

# Clean up
USER root
RUN rm -Rf {PAYARA_PATH}/temp

# Override the default Entrypoint with a basic delay_start script to give the database server time to initialize.
ENTRYPOINT ["/bin/sh", "-c", "${PAYARA_PATH}/init_2_delay_start.sh && ${PAYARA_PATH}/bin/startInForeground.sh --passwordfile=/opt/pwdfile --postbootcommandfile ${POSTBOOT_COMMANDS} ${PAYARA_DOMAIN}"]
