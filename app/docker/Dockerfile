FROM registry.redhat.io/jboss-webserver-5/jws57-openjdk11-openshift-rhel8:5.7.0-6

# Elevate
USER root

# Add war file (could bet a get to nexus repo)
ADD app/bin/sample.war /deployments/sample.war
RUN chmod 640 /deployments/sample.war && chown 185:root /deployments/sample.war

# JWS User
USER 185




