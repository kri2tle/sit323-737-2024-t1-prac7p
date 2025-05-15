# sit323-737-2024-t1-prac7p

MongoDB Integration with Kubernetes Application
This repository demonstrates how to integrate MongoDB with a Node.js application deployed in Kubernetes, as part of the SIT323/SIT737 Cloud Native Application Development course.
Table of Contents

Prerequisites
Project Structure
Setup Instructions
Testing the Application
Implementation Details
Troubleshooting
Screenshots

Prerequisites

Git (https://github.com)
Visual Studio Code (https://code.visualstudio.com/)
Node.js (https://nodejs.org/en/download/)
Docker Desktop with Kubernetes enabled
kubectl (command-line tool for Kubernetes)
MongoDB client (optional, for testing)

Project Structure
sit323-737-2024-t1-prac7p/
├── kubernetes/
│   ├── mongodb/
│   │   ├── mongodb-deployment.yaml   # MongoDB deployment configuration
│   │   ├── mongodb-service.yaml      # MongoDB service configuration
│   │   └── mongodb-secret.yaml       # Secret for MongoDB credentials
│   └── app/
│       ├── app-deployment.yaml       # Application deployment configuration
│       └── app-service.yaml          # Application service configuration
├── src/
│   ├── app.js                        # Main application file with API endpoints
│   ├── db.js                         # MongoDB connection module
│   └── package.json                  # Node.js dependencies
├── Dockerfile                        # Container definition for the application
└── README.md                         # This documentation file
Setup Instructions
1. Clone the Repository
bashgit clone https://github.com/your-username/sit323-737-2024-t1-prac7p.git
cd sit323-737-2024-t1-prac7p
2. Create the Required Kubernetes Resources
Create MongoDB Secret
bashkubectl create secret generic mongodb-secret --from-literal=username=admin --from-literal=password=password123
Deploy MongoDB
bashkubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:latest
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: password
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
      volumes:
      - name: mongodb-data
        emptyDir: {}
EOF
Create MongoDB Service
bashkubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
  type: ClusterIP
EOF
3. Build and Deploy the Application
Create the Node.js Application Files
First, create the src directory and required files:
bashmkdir -p src
Create the DB Connection Module (db.js)
javascript// src/db.js
const { MongoClient } = require('mongodb');

// MongoDB connection parameters from environment variables
const username = process.env.MONGO_USERNAME;
const password = process.env.MONGO_PASSWORD;
const host = process.env.MONGO_HOST || 'mongodb-service';
const port = process.env.MONGO_PORT || '27017';
const database = process.env.MONGO_DB || 'myapp';

// MongoDB connection URI
const uri = `mongodb://${username}:${password}@${host}:${port}/${database}?authSource=admin`;

let client;
let dbConnection;

module.exports = {
  // Connect to MongoDB
  connectToDatabase: async () => {
    try {
      client = new MongoClient(uri);
      await client.connect();
      dbConnection = client.db(database);
      console.log('Successfully connected to MongoDB');
      return dbConnection;
    } catch (error) {
      console.error('Error connecting to MongoDB:', error);
      throw error;
    }
  },
  
  // Get the database connection
  getDb: () => dbConnection,
  
  // Close the database connection
  closeConnection: async () => {
    if (client) {
      await client.close();
      console.log('MongoDB connection closed');
    }
  }
};
Create the Express Application (app.js)
javascript// src/app.js
const express = require('express');
const { connectToDatabase, getDb, closeConnection } = require('./db');

const app = express();
app.use(express.json());

let db;

// Connect to MongoDB before starting the server
connectToDatabase()
  .then((database) => {
    db = database;
    // Start server
    const PORT = process.env.PORT || 3000;
    app.listen(PORT, () => {
      console.log(`Server running on port ${PORT}`);
    });
  })
  .catch((err) => {
    console.error('Failed to connect to the database:', err);
    process.exit(1);
  });

// Basic route for testing
app.get('/', (req, res) => {
  res.json({ message: 'Welcome to MongoDB Kubernetes Integration Example' });
});

// CRUD Operations
// Create - Add a new item
app.post('/items', async (req, res) => {
  try {
    const newItem = req.body;
    const result = await db.collection('items').insertOne(newItem);
    res.status(201).json({ id: result.insertedId, ...newItem });
  } catch (error) {
    console.error('Error creating item:', error);
    res.status(500).json({ error: 'Failed to create item' });
  }
});

// Read - Get all items
app.get('/items', async (req, res) => {
  try {
    const items = await db.collection('items').find({}).toArray();
    res.status(200).json(items);
  } catch (error) {
    console.error('Error fetching items:', error);
    res.status(500).json({ error: 'Failed to fetch items' });
  }
});

// Read - Get a specific item
app.get('/items/:id', async (req, res) => {
  try {
    const { ObjectId } = require('mongodb');
    const id = new ObjectId(req.params.id);
    const item = await db.collection('items').findOne({ _id: id });
    
    if (!item) {
      return res.status(404).json({ error: 'Item not found' });
    }
    
    res.status(200).json(item);
  } catch (error) {
    console.error('Error fetching item:', error);
    res.status(500).json({ error: 'Failed to fetch item' });
  }
});

// Update - Update an item
app.put('/items/:id', async (req, res) => {
  try {
    const { ObjectId } = require('mongodb');
    const id = new ObjectId(req.params.id);
    const updatedItem = req.body;
    
    const result = await db.collection('items').updateOne(
      { _id: id },
      { $set: updatedItem }
    );
    
    if (result.matchedCount === 0) {
      return res.status(404).json({ error: 'Item not found' });
    }
    
    res.status(200).json({ id: req.params.id, ...updatedItem });
  } catch (error) {
    console.error('Error updating item:', error);
    res.status(500).json({ error: 'Failed to update item' });
  }
});

// Delete - Delete an item
app.delete('/items/:id', async (req, res) => {
  try {
    const { ObjectId } = require('mongodb');
    const id = new ObjectId(req.params.id);
    
    const result = await db.collection('items').deleteOne({ _id: id });
    
    if (result.deletedCount === 0) {
      return res.status(404).json({ error: 'Item not found' });
    }
    
    res.status(200).json({ message: 'Item deleted successfully' });
  } catch (error) {
    console.error('Error deleting item:', error);
    res.status(500).json({ error: 'Failed to delete item' });
  }
});

// Graceful shutdown
process.on('SIGINT', async () => {
  await closeConnection();
  process.exit(0);
});
Create package.json
json{
  "name": "mongodb-kubernetes-integration",
  "version": "1.0.0",
  "description": "MongoDB integration with Kubernetes",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [
    "mongodb",
    "kubernetes",
    "nodejs"
  ],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.18.2",
    "mongodb": "^6.3.0"
  }
}
Install the dependencies:
bashcd src
npm install
cd ..
Create Dockerfile
dockerfileFROM node:16

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
COPY src/package*.json ./
RUN npm install

# Bundle app source
COPY src/ .

# Expose the port the app runs on
EXPOSE 3000

# Command to run the application
CMD ["npm", "start"]
Build and Push the Docker Image
bashdocker build -t yourusername/mongodb-kubernetes-app:latest .
docker push yourusername/mongodb-kubernetes-app:latest
⚠️ Replace yourusername with your actual Docker Hub username.
Deploy the Application
Create the app deployment:
bashkubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: yourusername/mongodb-kubernetes-app:latest
        ports:
        - containerPort: 3000
        env:
        - name: MONGO_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: username
        - name: MONGO_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: password
        - name: MONGO_HOST
          value: "mongodb-service"
        - name: MONGO_PORT
          value: "27017"
        - name: MONGO_DB
          value: "myapp"
EOF
⚠️ Replace yourusername with your actual Docker Hub username.
Create the app service:
bashkubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 3000
  type: NodePort
EOF
4. Verify Deployment
Check if all pods are running:
bashkubectl get pods
Check services:
bashkubectl get services
Testing the Application
Access the Application
Get the NodePort for your application:
bashkubectl get service myapp-service
Look for the NodePort (e.g., 80:30123/TCP) and access the application at:
http://localhost:30123
Replace 30123 with your actual NodePort.
Alternatively, use port forwarding:
bashkubectl port-forward service/myapp-service 8080:80
Then access at: http://localhost:8080
Test CRUD Operations
Create an Item
bashcurl -X POST -H "Content-Type: application/json" -d '{"name": "Test Item", "description": "This is a test"}' http://localhost:8080/items
Get All Items
bashcurl http://localhost:8080/items
Get a Specific Item
bashcurl http://localhost:8080/items/[ITEM_ID]
Replace [ITEM_ID] with the actual ID returned from the create operation.
Update an Item
bashcurl -X PUT -H "Content-Type: application/json" -d '{"name": "Updated Item", "description": "This item has been updated"}' http://localhost:8080/items/[ITEM_ID]
Delete an Item
bashcurl -X DELETE http://localhost:8080/items/[ITEM_ID]
Implementation Details
MongoDB Configuration

MongoDB is deployed as a single instance within the Kubernetes cluster
Authentication is enabled with credentials stored in a Kubernetes Secret
Data is stored in an emptyDir volume (note: this is temporary and will be lost if the pod is deleted)
The database is exposed only within the cluster via a ClusterIP service

Application Design

RESTful API built with Express.js
Full CRUD operations for a simple "items" collection
MongoDB connection management with proper error handling
Graceful shutdown handling for clean database disconnection

Security Considerations

Credentials are stored in Kubernetes Secrets rather than hardcoded
MongoDB is not exposed outside the cluster
Environment variables are used for configuration

Troubleshooting
Common Issues and Solutions

MongoDB connection errors:

Ensure MongoDB pod is running (kubectl get pods)
Check MongoDB service is created (kubectl get services)
Verify the secret has correct credentials


ImagePullBackOff errors:

Make sure you're logged into Docker Hub (docker login)
Check if the image exists in your repository
Verify the image name in your deployment is correct


Pod scheduling issues:

For Docker Desktop, using emptyDir instead of PVC resolves most scheduling conflicts


Application crashes:

Check logs for errors: kubectl logs deployment/myapp
Verify environment variables are correct
