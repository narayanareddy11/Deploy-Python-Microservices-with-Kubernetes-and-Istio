Here’s the complete file structure and code formatted as a `README.md` file. You can copy and paste this into a `README.md` file in your project repository.

---

```markdown
# E-Commerce Microservices with Kafka

This project demonstrates a simple e-commerce application with three microservices:
1. **Product Service**: Displays products.
2. **Cart Service**: Manages the shopping cart.
3. **Order Service**: Handles order placement.

Kafka is used for event-driven communication between the microservices.

---

## File Structure

```
e-commerce-microservices/
├── product-service/
│   ├── app.py
│   ├── requirements.txt
│   ├── templates/
│   │   └── products.html
│   └── Dockerfile
├── cart-service/
│   ├── app.py
│   ├── requirements.txt
│   ├── templates/
│   │   └── cart.html
│   └── Dockerfile
├── order-service/
│   ├── app.py
│   ├── requirements.txt
│   ├── templates/
│   │   └── order.html
│   └── Dockerfile
├── kubernetes/
│   ├── product-service-deployment.yaml
│   ├── cart-service-deployment.yaml
│   ├── order-service-deployment.yaml
│   └── ingress.yaml
└── README.md
```

---

## Code

### 1. Product Service

#### `product-service/app.py`
```python
from flask import Flask, render_template

app = Flask(__name__)

products = [
    {"id": 1, "name": "Laptop", "price": 999.99},
    {"id": 2, "name": "Smartphone", "price": 499.99},
    {"id": 3, "name": "Headphones", "price": 149.99},
]

@app.route("/")
def view_products():
    return render_template("products.html", products=products)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

#### `product-service/requirements.txt`
```
Flask==2.3.2
```

#### `product-service/Dockerfile`
```Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

---

### 2. Cart Service (Kafka Consumer)

#### `cart-service/app.py`
```python
from flask import Flask, render_template, request, redirect
from confluent_kafka import Consumer, KafkaException
import json
import threading

app = Flask(__name__)
cart = []

# Kafka Consumer Setup
conf = {'bootstrap.servers': 'kafka:9092', 'group.id': 'cart-service', 'auto.offset.reset': 'earliest'}
consumer = Consumer(conf)
consumer.subscribe(["order_placed"])

def kafka_consumer():
    while True:
        msg = consumer.poll(1.0)
        if msg is None:
            continue
        if msg.error():
            raise KafkaException(msg.error())
        else:
            cart.clear()

# Start Kafka thread
thread = threading.Thread(target=kafka_consumer)
thread.daemon = True
thread.start()

@app.route("/")
def view_cart():
    return render_template("cart.html", cart=cart)

@app.route("/add", methods=["POST"])
def add_to_cart():
    product_id = int(request.form["product_id"])
    product_name = request.form["product_name"]
    product_price = float(request.form["product_price"])
    cart.append({"id": product_id, "name": product_name, "price": product_price})
    return redirect("/")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

#### `cart-service/requirements.txt`
```
Flask==2.3.2
confluent-kafka==2.2.0
```

#### `cart-service/Dockerfile`
```Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

---

### 3. Order Service (Kafka Producer)

#### `order-service/app.py`
```python
from flask import Flask, render_template, request
from confluent_kafka import Producer
import json

app = Flask(__name__)

# Kafka Producer Setup
conf = {'bootstrap.servers': 'kafka:9092', 'client.id': 'order-service'}
producer = Producer(conf)

@app.route("/")
def view_order():
    return render_template("order.html")

@app.route("/place_order", methods=["POST"])
def place_order():
    name = request.form["name"]
    address = request.form["address"]
    order_data = {"name": name, "address": address}
    producer.produce("order_placed", json.dumps(order_data).encode('utf-8'))
    producer.flush()
    return f"Order placed for {name} to {address}. Thank you!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

#### `order-service/requirements.txt`
```
Flask==2.3.2
confluent-kafka==2.2.0
```

#### `order-service/Dockerfile`
```Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

---

### 4. Kubernetes Files

#### `kubernetes/product-service-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
      - name: product-service
        image: product-service:latest
        ports:
        - containerPort: 5000
```

#### `kubernetes/cart-service-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cart-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cart-service
  template:
    metadata:
      labels:
        app: cart-service
    spec:
      containers:
      - name: cart-service
        image: cart-service:latest
        ports:
        - containerPort: 5000
        env:
        - name: KAFKA_BOOTSTRAP_SERVERS
          value: "kafka:9092"
```

#### `kubernetes/order-service-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: order-service:latest
        ports:
        - containerPort: 5000
        env:
        - name: KAFKA_BOOTSTRAP_SERVERS
          value: "kafka:9092"
```

---

## How to Run

1. **Deploy Kafka in Kubernetes**:
   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm install kafka bitnami/kafka
   ```

2. **Build Docker Images**:
   ```bash
   docker build -t product-service:latest ./product-service
   docker build -t cart-service:latest ./cart-service
   docker build -t order-service:latest ./order-service
   ```

3. **Deploy to Kubernetes**:
   ```bash
   kubectl apply -f kubernetes/
   ```

4. **Access the Application**:
   - Use `kubectl port-forward` to access the services locally.
   - Alternatively, set up an Ingress controller.

---

## License
This project is licensed under the MIT License.
```

---

### **Steps to Use**
1. Copy the content above into a `README.md` file.
2. Save the file in the root of your project directory.
3. Push the repository to GitHub or share it as needed.

Let me know if you need further assistance! 😊
