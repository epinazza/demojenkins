# Start from WSO2 Micro Integrator base image
FROM wso2/wso2mi:4.4.0

# Set working directory to where WSO2 deploys CAR files
WORKDIR /home/wso2carbon/wso2mi-4.4.0/repository/deployment/server/carbonapps/

# Copy your CAR file into the deployment folder
COPY ./AppointmentServices_1.0.0.car ./

# Expose API and management ports
EXPOSE 8290 8253

# Start Micro Integrator when container runs
CMD ["/home/wso2carbon/wso2mi-4.4.0/bin/micro-integrator.sh"]
