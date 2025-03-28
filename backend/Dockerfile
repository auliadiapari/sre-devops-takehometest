# # Base image
# # FROM node:18

FROM --platform=linux/amd64 node:16-alpine

# Create app directory
WORKDIR /usr/src/app

# A wildcard is used to ensure both package.json AND package-lock.json are copied
COPY package*.json ./

# Install app dependencies
RUN npm install

# Bundle app source
COPY . .

# Creates a "dist" folder with the production build
RUN npm run build

# Expose the port on which the app will run
EXPOSE 3000

# Start the server using the production build
CMD ["npm", "run", "start:prod"]



# # Use an official Node.js runtime as a parent image
# FROM --platform=linux/amd64 node:18-bookworm

# # Set the working directory in the container
# WORKDIR /app

# # Copy package.json and package-lock.json to the working directory
# COPY package*.json ./

# # Install application dependencies
# RUN npm install

# # Copy the rest of the application files to the working directory
# COPY . .

# # Build the NestJS application
# RUN npm run build

# # Expose the port your NestJS application runs on (default is often 3000)
# EXPOSE 3000

# # Command to run the application
# CMD [ "npm", "run", "start" ]