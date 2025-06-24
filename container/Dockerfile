# Use Red Hat's Universal Base Image for Nginx
FROM registry.access.redhat.com/ubi8/nginx-120

USER root
# Create the directory for .well-known
# This is crucial for OpenID Connect discovery and other web standards
RUN mkdir -p /etc/nginx/html/.well-known/
RUN chown -R nginx:nginx /etc/nginx/html

USER nginx

# Copy the keys.json file into the Nginx web root
# This file typically contains public keys for verifying digital signatures
COPY keys.json /etc/nginx/html/

# Copy the openid-configuration file into the .well-known directory
# This file is used for OpenID Connect discovery
COPY openid-configuration /etc/nginx/html/.well-known/


# Expose port 80, which is the default HTTP port Nginx listens on
EXPOSE 80

# Command to run Nginx in the foreground
# This keeps the container running
CMD ["nginx", "-g", "daemon off;"]
