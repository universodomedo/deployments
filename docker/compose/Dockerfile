# Use the official Node.js image as the base image
FROM node:19

# Set the working directory in the container
WORKDIR /usr/src/app

# Copy package.json and package-lock.json (or yarn.lock) to the container
COPY package*.json ./

# Install the project dependencies
RUN npm install
RUN npm install -g typescript

# Copy the rest of the application code to the container
COPY . .

# Expose the port on which the Vite development server will run
EXPOSE 5173

# Define the command to run the application
CMD ["npm", "run", "dev", "--host"]
