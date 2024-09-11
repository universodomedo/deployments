# Use the official Node.js image from Docker Hub
FROM node:14

# Create and change to the app directory
WORKDIR /usr/src/app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install app dependencies
RUN npm install

# Copy the rest of the app's source code
COPY . .

# Build the app
RUN npm run build

# Expose the port the app runs on
EXPOSE 3030

# Define the command to run the app
CMD ["npm", "start"]
