# Use Red Hat's Univeral Base Image for nginx
FROM registry.access.redhat.com/ubi8/nginx-120

USER root
# Create the directory for .well-known
# This is crucial for OpenID Connect discovery and other web standards
RUN mkdir -p /etc/nginx/html/.well-known/
RUN chown -R nginx:nginx /etc/nginx/html

# Copy your custom Nginx configuration file
# This will override or supplement the default config depending on the base image
COPY nginx.conf /etc/nginx/nginx.conf
RUN chown -R nginx:nginx /etc/nginx/nginx.conf
RUN chown -R nginx:nginx /var/log/nginx
RUN chown -R nginx:nginx /var/lib/nginx/
RUN chown -R nginx:nginx /var/run/
USER nginx



# Copy the keys.json file into the Nginx web root
# This file typically contains public keys for verifying digital signatures
COPY keys.json /etc/nginx/html/

# Copy the openid-configuration file into the .well-known directory
# This file is used for OpenID Connect discovery
COPY openid-configuration /etc/nginx/html/.well-known/


# Expose port 8080, which is the default HTTP port Nginx listens on
EXPOSE 8080

# Command to run Nginx in the foreground
# This keeps the container running
CMD ["nginx", "-g", "daemon off;"]
