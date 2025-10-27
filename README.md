#Setup Google Spanner
Project Structure

<img width="800" height="318" alt="image" src="https://github.com/user-attachments/assets/0f4a2a13-b416-4403-a38e-88dce0f90e6f" />


## 1. Prerequisites
   - Enable the following APIs in your GCP project:
      - Compute Engine API
      - Kubernetes Engine API
      - Cloud Spanner API
      - Cloud SQL Admin API
      - Artifact Registry API (to store Docker images)

  - Initial Configuration: 
```ruby
export PROJECT_ID="[YOUR_GCP_PROJECT_ID]"
export REGION_BE="europe-west1"
export REGION_SY="australia-southeast1"
export REGION_SG="asia-southeast1"
export GSA_NAME="benchmarking-gsa"
export KSA_NAME="benchmarking-ksa"

gcloud config set project ${PROJECT_ID}
```

## 2. GKE clusters setup
   a. Create the cluster in Belgium 
```ruby
gcloud container clusters create belgium-cluster \
    --region=${REGION_BE} \
    --num-nodes=1 \
    --machine-type=e2-medium \
    --scopes=cloud-platform \
    --workload-pool=${PROJECT_ID}.svc.id.goog
```
   b. Create the cluster in Sydney
```ruby
gcloud container clusters create sydney-cluster \
    --region=${REGION_SY} \
    --num-nodes=1 \
    --machine-type=e2-medium \
    --scopes=cloud-platform \
    --workload-pool=${PROJECT_ID}.svc.id.goog
```

## 3. Create service accounts and IAM bindings
   a. Create a GCP Service Account (GSA) for our app
```ruby
gcloud iam service-accounts create ${GSA_NAME} \
    --display-name="Service Account for Benchmarking App"
```
   b. Grant the GSA the necessary Spanner role
```ruby
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/cloudsql.client"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/spanner.databaseUser"
```
   c. Allow the KSA to impersonate the GSA
```ruby
gcloud iam service-accounts add-iam-policy-binding ${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
    --role="roles/iam.workloadIdentityUser" \
    --member="serviceAccount:${PROJECT_ID}.svc.id.goog[default/${KSA_NAME}]"
```

## 4. Database Setup
### Part 1: Spanner
a. Create Spanner instance: 
```ruby
gcloud spanner instances create "spanner-instance-singapore" \
    --config=regional-${REGION_SG} \
    --processing-units=100 \
    --edition=STANDARD
```
b. Create spanner database
```ruby
gcloud spanner databases create "latency_test_db" \
    --instance="spanner-instance-singapore" \
    --database-dialect=POSTGRESQL
```
  c. Create Data Schema:
```ruby
CREATE TABLE product_catalog (
    product_id   VARCHAR(36) NOT NULL,
    product_name VARCHAR(255),
    description  VARCHAR(1024),
    price        NUMERIC,
    PRIMARY KEY (product_id)
);

CREATE TABLE orders (
    order_id   VARCHAR(36) NOT NULL,
    product_id VARCHAR(36),
    quantity   BIGINT,
    order_date TIMESTAMPTZ,
    PRIMARY KEY (order_id)
);
```
  d. Insert dummy data:
```ruby
INSERT INTO product_catalog (product_id, product_name, description, price)
VALUES
('1', 'Laptop', 'High-performance laptop', 1200.50),
('2', 'Smartphone', 'Latest model smartphone', 800.00),
('3', 'Headphones', 'Noise-cancelling headphones', 250.75),
('4', 'Keyboard', 'Mechanical keyboard', 150.00),
('5', 'Mouse', 'Wireless gaming mouse', 75.50),
('6', 'Monitor', '4K Ultra HD monitor', 600.00),
('7', 'Webcam', '1080p HD webcam', 90.25),
('8', 'Desk Chair', 'Ergonomic office chair', 350.00),
('9', 'Desk Lamp', 'LED desk lamp', 40.00),
('10', 'Backpack', 'Waterproof laptop backpack', 60.50),
('11', 'Coffee Maker', 'Drip coffee maker', 80.00),
('12', 'Blender', 'High-speed blender', 120.75),
('13', 'Toaster', '4-slice toaster', 50.00),
('14', 'Microwave', 'Countertop microwave oven', 150.50),
('15', 'Router', 'Wi-Fi 6 router', 200.00),
('16', 'External Hard Drive', '2TB portable hard drive', 70.25),
('17', 'Printer', 'All-in-one wireless printer', 180.00),
('18', 'Notebook', 'Spiral bound notebook', 5.00),
('19', 'Pen Set', 'Set of 12 gel pens', 12.00),
('20', 'Stapler', 'Standard office stapler', 15.00);
```
```ruby
INSERT INTO orders (order_id, product_id, quantity, order_date)
VALUES
('45821', '1', 1, '2025-09-23T10:00:00Z'),
('98322', '2', 2, '2025-09-23T10:05:00Z'),
('12093', '3', 1, '2025-09-23T10:10:00Z'),
('77454', '4', 3, '2025-09-23T10:15:00Z'),
('30285', '5', 1, '2025-09-23T10:20:00Z'),
('65236', '10', 1, '2025-09-23T10:25:00Z'),
('88127', '11', 2, '2025-09-23T10:30:00Z'),
('23908', '12', 1, '2025-09-23T10:35:00Z'),
('51789', '13', 3, '2025-09-23T10:40:00Z'),
('92450', '14', 1, '2025-09-23T10:45:00Z'),
('11561', '1', 2, '2025-09-23T10:50:00Z'),
('67342', '2', 1, '2025-09-23T10:55:00Z'),
('40013', '3', 3, '2025-09-23T11:00:00Z'),
('81234', '4', 1, '2025-09-23T11:05:00Z'),
('29985', '5', 2, '2025-09-23T11:10:00Z'),
('76546', '10', 1, '2025-09-23T11:15:00Z'),
('50027', '11', 3, '2025-09-23T11:20:00Z'),
('33878', '12', 1, '2025-09-23T11:25:00Z'),
('90129', '13', 2, '2025-09-23T11:30:00Z'),
('25670', '14', 1, '2025-09-23T11:35:00Z');
```
### Part 3: GKE App (Node.js/TypeORM)
1. Setup Project Directory
```
mkdir latency-app
cd latency-app
npm init -y
npm install express pg typeorm reflect-metadata
npm install -D typescript @types/express @types/node ts-node nodemon
```
2. Create Application Files
```
touch tsconfig.json .dockerignore Dockerfile
mkdir src
touch src/index.ts src/data-source.ts
mkdir src/entity
touch src/entity/product.ts src/entity/order.ts
```

>tsconfig.json
```ruby
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "es6",
    "lib": ["es6"],
    "moduleResolution": "node",
    "rootDir": "src",
    "outDir": "dist",
    "sourceMap": true,
    "esModuleInterop": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "strict": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```
>src/entity/product.ts (Defines the ProductCatalog table)
```ruby
import { Entity, PrimaryColumn, Column, BaseEntity } from "typeorm";

@Entity('product_catalog')
export class Product extends BaseEntity {
    @PrimaryColumn({ type: 'varchar' })
    product_id!: string;

    @Column({ type: 'varchar' })
    product_name!: string;

    @Column({ type: 'varchar' })
    description!: string;

    @Column({ type: 'float' })
    price!: number;
}
```
>src/entity/order.ts (Defines the Orders table)
```ruby
import { Entity, PrimaryColumn, Column, BaseEntity } from "typeorm";

@Entity('orders')
export class Order extends BaseEntity {
    @PrimaryColumn({ type: 'varchar' })
    order_id!: string;

    @Column({ type: 'varchar' })
    product_id!: string;

    @Column({ type: 'int' })
    quantity!: number;

    @Column({ type: 'timestamptz' })
    order_date!: Date;
}
```
>src/data-source.ts (Configures the database connection)
```ruby
import "reflect-metadata";
import { DataSource } from "typeorm";
import { Product } from "./entity/product";
import { Order } from "./entity/order";

export const AppDataSource = new DataSource({
    type: "postgres",
    host: process.env.DB_HOST || "localhost",
    port: parseInt(process.env.DB_PORT || "5432"),
    username: process.env.DB_USER || "postgres",
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME || "latency-db",
    synchronize: false, // We already created tables
    logging: true,
    entities: [Product, Order],
    migrations: [],
    subscribers: [],
});
```
>src/index.ts (The main application logic with API endpoints)
```ruby
import express from 'express';
import { AppDataSource } from './data-source';
import { Product } from './entity/product'; 
import { Order } from './entity/order';  

const app = express();
app.use(express.json());
const PORT = process.env.PORT || 8080;

// Initialize the database connection first
AppDataSource.initialize()
    .then(() => {
        // This block runs ONLY after the connection is successful
        console.log("Database connection established!");

        // --- DEFINE ALL API ENDPOINTS HERE ---

        // READ function: Get a product by its ID
        app.get('/products/:id', async (req, res) => {
            try {
                const product = await Product.findOneBy({ product_id: req.params.id });
                if (product) {
                    res.json(product);
                } else {
                    res.status(404).send('Product not found');
                }
            } catch (err) {
                console.error(err);
                res.status(500).send('Server Error');
            }
        });

        // WRITE function: Update an order's quantity
        app.put('/orders/:id', async (req, res) => {
            try {
                const { quantity } = req.body;
                if (typeof quantity !== 'number') {
                    return res.status(400).send('Invalid quantity provided');
                }
                const order = await Order.findOneBy({ order_id: req.params.id });
                if (order) {
                    order.quantity = quantity;
                    await order.save();
                    res.json(order);
                } else {
                    res.status(404).send('Order not found');
                }
            } catch (err) {
                console.error(err);
                res.status(500).send('Server Error');
            }
        });

        app.get('/health', (req, res) => {
          res.status(200).send('OK');
        });

        // --- START THE SERVER LAST ---
        // Now that the DB is connected and routes are set up, start the server.
        app.listen(PORT, () => {
            console.log(`Server is running on port ${PORT}`);
        });

    })
    .catch((error) => {
        // This block runs if the initial connection fails
        console.error("DB Error: Could not connect to the database", error);
    });
```
>Dockerfile
```ruby
# ---- Base Stage ----
FROM node:18-slim AS base
WORKDIR /app
COPY package*.json ./
RUN npm install

# ---- Build Stage ----
FROM base AS build
WORKDIR /app
COPY --from=base /app/node_modules ./node_modules
COPY . .
RUN npm install -g typescript
RUN tsc

# ---- Production Stage ----
FROM node:18-slim AS production
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
CMD ["node", "dist/index.js"]
```
3. Build and Push the Docker Image
```ruby
# Create a Docker repository in Artifact Registry
gcloud artifacts repositories create benchmark-repo \
    --repository-format=docker \
    --location=${REGION_SG} \
    --description="Repo for latency test app"
```
```ruby
# Configure Docker to authenticate with Artifact Registry
gcloud auth configure-docker ${REGION_SG}-docker.pkg.dev
```
```ruby
# Build and push the image using Cloud Build (easiest way)
# This command uses the Dockerfile in your current directory
gcloud builds submit --tag ${REGION_SG}-docker.pkg.dev/${PROJECT_ID}/benchmark-repo/latency-app:v1
```

## 5. Deploy to GKE
### Test Spanner (Sidecar deployment pattern)
1. Deployment for Spanner with PGAdapter (Sidecar)
```
touch spanner-app-sidecar.yaml
```
>spanner-app-sidecar.yaml
```ruby
# 1. Kubernetes Service Account (KSA) with the critical annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: benchmarking-ksa
  annotations:
    # This annotation links the KSA to your GSA for permissions
    iam.gke.io/gcp-service-account: benchmarking-gsa@${PROJECT_ID}.iam.gserviceaccount.com
---
# 2. Deployment for your application and the sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: latency-app-deployment-spanner-sidecar
spec:
  replicas: 2
  selector:
    matchLabels:
      app: latency-app-spanner-sidecar
  template:
    metadata:
      labels:
        app: latency-app-spanner-sidecar
    spec:
      # This tells the pod to use the KSA we defined above
      serviceAccountName: benchmarking-ksa
      containers:
      - name: app
        image: ${REGION_BE}-docker.pkg.dev/${PROJECT_ID}/benchmark-repo/latency-app:v1
        ports:
        - containerPort: 8080
        startupProbe:
          httpGet:
            path: /health
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
        env:
          - name: DB_NAME
            value: "latency_test_db"
          - name: DB_HOST
            value: "127.0.0.1"
          - name: DB_PORT
            value: "5432"
      # This is the PGAdapter sidecar container
      - name: pgadapter
        image: gcr.io/cloud-spanner-pg-adapter/pgadapter
        ports:
        - containerPort: 5432
        args:
          - "-p"
          - "${PROJECT_ID}"
          - "-i"
          - "spanner-instance-belgium"
          - "-d"
          - "latency_test_db"
          - "-s"
          - "5432"
---
# 3. LoadBalancer Service to expose the application
apiVersion: v1
kind: Service
metadata:
  name: latency-app-service-spanner-sidecar
spec:
  type: LoadBalancer
  selector:
    app: latency-app-spanner-sidecar
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080

```
2. Deploy to Belgium and Sydney Clusters:
   a. get Sydney-cluster credentials:
```
gcloud container clusters get-credentials sydney-cluster --region australia-southeast1
```
   b. Deploy to Sydney:
```ruby
kubectl apply -f spanner-app-sidecar.yaml
```
   c. Repeat for Belgium-cluster:
```ruby
gcloud container clusters get-credentials belgium-cluster --region europe-west1
kubectl apply -f spanner-app-sidecar.yaml
```
   d. To get Load Balancer IPs (You'll need this for testing):
```ruby
kubectl get service latency-app-service-spanner-sidecar
```

## 6. Create the Latency Test Script
You can now run the latency test, create script on local machine to simulate the client workload.

```
sudo apt-get update
sudo apt-get install -y nodejs npm
npm install axios
nano test-latency.js
```

>test-latency.js
```ruby
const axios = require('axios');

// --- CONFIGURATION ---
const BASE_URL = process.argv[2]; 
if (!BASE_URL) {
    console.error("Usage: node test-latency.js <URL>");
    process.exit(1);
}

const PRODUCT_IDS_TO_READ = ['1', '2', '3'];
const ORDER_TO_UPDATE = { id: '45821', newQuantity: 5 };
const ITERATIONS = 10; 

// --- HELPER FUNCTIONS ---
const getProduct = (productId) => axios.get(`${BASE_URL}/products/${productId}`);
const updateOrder = (orderId, quantity) => axios.put(`${BASE_URL}/orders/${orderId}`, { quantity });

// --- TEST CYCLES ---
const runReadCycle = async () => {
    console.time("Read Latency (3 ops)");
    await getProduct(PRODUCT_IDS_TO_READ[0]);
    await getProduct(PRODUCT_IDS_TO_READ[1]);
    await getProduct(PRODUCT_IDS_TO_READ[2]);
    console.timeEnd("Read Latency (3 ops)");
};

const runWriteCycle = async () => {
    console.time("Write Latency (1 op)");
    await updateOrder(ORDER_TO_UPDATE.id, ORDER_TO_UPDATE.newQuantity);
    console.timeEnd("Write Latency (1 op)");
};

const runCombinedCycle = async () => {
    console.time("Combined Latency (3 reads, 1 write)");
    await getProduct(PRODUCT_IDS_TO_READ[0]);
    await getProduct(PRODUCT_IDS_TO_READ[1]);
    await getProduct(PRODUCT_IDS_TO_READ[2]);
    await updateOrder(ORDER_TO_UPDATE.id, ORDER_TO_UPDATE.newQuantity);
    console.timeEnd("Combined Latency (3 reads, 1 write)");
};

// --- MAIN EXECUTION ---
const main = async () => {
    console.log(`\n--- Starting Full Latency Test Suite on ${BASE_URL} ---`);
    
    // --- Warm-up Run ---
    console.log("\nPerforming one warm-up run...");
    await runCombinedCycle();
    console.log("Warm-up complete.");

    // --- READ TEST ---
    console.log(`\n--- Running READ Test (${ITERATIONS} iterations) ---`);
    for (let i = 1; i <= ITERATIONS; i++) {
        process.stdout.write(`Iteration ${i}/${ITERATIONS}... `);
        await runReadCycle();
    }

    // --- WRITE TEST ---
    console.log(`\n\n--- Running WRITE Test (${ITERATIONS} iterations) ---`);
    for (let i = 1; i <= ITERATIONS; i++) {
        process.stdout.write(`Iteration ${i}/${ITERATIONS}... `);
        await runWriteCycle();
    }
    
    // --- COMBINED TEST ---
    console.log(`\n\n--- Running COMBINED Test (${ITERATIONS} iterations) ---`);
    for (let i = 1; i <= ITERATIONS; i++) {
        process.stdout.write(`Iteration ${i}/${ITERATIONS}... `);
        await runCombinedCycle();
    }

    console.log("\n\n--- All Tests Complete ---");
};

main().catch(err => {
    console.error("\nAn error occurred during the test:", err.message);
    if (err.response) {
        console.error("Response Status:", err.response.status);
        console.error("Response Data:", err.response.data);
    }
});
```
```
node test-latency.js http://[Change_to_the_IP_address_that_you_copied]
```


