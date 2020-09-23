# Example-Python-Docker-Iothub-PostgreSQL

- [1. Introduction](#1-introduction)
- [2. Before You Start](#2-before-you-start)
- [3. Downloading the Project](#3-downloading-the-project)
- [4. Deploy the app to WISE-PaaS](#4-deploy-the-app-to-wise-paas)
- [5. Application Introduce](#5-application-introduce)
  - [5-1. index.py](#5-1-indexpy)
  - [5-2. publisher.py](#5-2-publisherpy)
- [6. Kubernetes Config](#6-kubernetes-config)
  - [6-1. deployment.yaml](#6-1-deploymentyaml)
  - [6-2. ingress.yaml](#6-2-ingressyaml)
  - [6-3. service.yaml](#6-3-serviceyaml)
- [7. Docker](#7-docker)
  - [7-1. dockerfile](#7-1-dockerfile)
- [8. Deployment Application Steps](#8-deployment-application-steps)
  - [8-1. build Docker image](#8-1-build-docker-image)
  - [8-2. push it to Docker Hub](#8-2-push-it-to-docker-hub)
  - [8-3. create kubernetes object ( All object are in the k8s folder)](#8-3-create-kubernetes-object--all-object-are-in-the-k8s-folder)
  - [8-4. Check（Pod status is running for success）](#8-4-checkpod-status-is-running-for-success)
  - [8-5. Send message to wise-paas by MQTT](#8-5-send-message-to-wise-paas-by-mqtt)
  - [8-6. Check that postgresql has received data](#8-6-check-that-postgresql-has-received-data)

## 1. Introduction

This sample code shows how to deploy an application to the EnSaaS 4.0 environment and connect to the the database (Postgresql) and message broker (RabbitMQ) services provided by the platform.

## 2. Before You Start

1. Create a [Docker](https://www.docker.com/get-started) Account
2. Development Environment
   - Install [Docker](https://docs.docker.com/install/)
   - Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
   - Install [Postgresql](https://www.postgresql.org/)

## 3. Downloading the Project

    git clone https://github.com/WISE-PaaS/example-py-docker-iothub-postgresql.git

## 4. Deploy the app to WISE-PaaS

WISE-PaaS has 2 types of data centers

**SA DataCenter**：[https://portal-mp-ensaas.sa.wise-paas.com](https://portal-mp-ensaas.sa.wise-paas.com/)

- **Cluster**：eks004
  - **Workspace**：adv-training
    - **Namespace**：level2

**HZ DataCenter**：[https://portal-mp-ensaas.hz.wise-paas.com.cn](https://portal-mp-ensaas.hz.wise-paas.com.cn/)

- **Cluster**：eks006
  - **Workspace**：advtraining
    - **Namespace**：level2

## 5. Application Introduce

### 5-1. index.py

Simply backend appliaction。

```py
app = Flask(__name__)

# port from cloud environment variable or localhost:3000
port = int(os.getenv("PORT", 3000))


@app.route('/', methods=['GET'])
def root():

    if(port == 3000):
        return 'py-docker-iothub-postgresql successful'
    elif(port == int(os.getenv("PORT"))):
        return render_template('index.html')
```

This is the postgresql and MQTT connect config code，`ENSAAS_SERVICES` can get the application environment in WISE-PaaS。

```py

#need to be same name in WISE-PaaS service name
IOTHUB_SERVICE_NAME = 'p-rabbitmq'
DB_SERVICE_NAME = 'postgresql'

# Get the environment variables
ENSAAS_SERVICES = os.getenv('ENSAAS_SERVICES')
ENSAAS_SERVICES_js = json.loads(ENSAAS_SERVICES)

# --- MQTT(rabbitmq) ---
credentials = ENSAAS_SERVICES_js[IOTHUB_SERVICE_NAME][0]['credentials']
mqtt_credential = credentials['protocols']['mqtt']

broker = mqtt_credential['host']
username = mqtt_credential['username'].strip()
password = mqtt_credential['password'].strip()
mqtt_port = mqtt_credential['port']

# # --- Postgresql ---
credentials = ENSAAS_SERVICES_js[DB_SERVICE_NAME][0]['credentials']
database_database = credentials['database']
database_username = credentials['username'].strip()
database_password = credentials['password'].strip()
database_port = credentials['port']
database_host = credentials['host']

POSTGRES = {
    'user': database_username,
    'password': database_password,
    'db': database_database,
    'host': database_host,
    'port': database_port,
}
```

Retrieve the secret, which iothub the secret contains

    # List all secrets in the namespace
    $ kubectl get secret --namespace=level2
    
    # Output the secret content
    $ kubectl get secret {secret_name} --namespace=level2 -o yaml

![-oyaml](https://tva1.sinaimg.cn/large/007S8ZIlgy1giz6sk4hb3j30ta0c3n5j.jpg)

Copy the decoded content and paste it into the editor, such as Visual Studio Code, and let the plugin prettify it. You can now inspect the structure and start to construct your code.

    # Decoding the secret
    $ kubectl get secret {secret_name} --namespace=level2 -o jsonpath="{.data.ENSAAS_SERVICES}" | base64 --decode; echo

![-ojsonpath](https://tva1.sinaimg.cn/large/007S8ZIlgy1giz88jndhjj30up04gtd9.jpg)

Copy the decoded content to vscode and Save as **json** format

**Notice**：the `DB_SERVICE_NAME` and `IOTHUB_SERVICE_NAME` need to be same name in **decode content instance name**。, Not the instance name in portal-service

![copyDataVS](https://tva1.sinaimg.cn/large/007S8ZIlgy1giz80nrjndj317k0rk10h.jpg)

![copyDataVS](https://tva1.sinaimg.cn/large/007S8ZIlgy1giz873372qj317l0hsjwm.jpg)

Create schema and table，when we bind the postgresql we need to tell the group for it。

```py
#schema table
schema = 'livingroom'
table = 'temperature'
#when we bind group need to tell postgresql service
group = 'groupfamily'
# connect to server
engine = sqlalchemy.create_engine('postgresql://%(user)s:\
%(password)s@%(host)s:%(port)s/%(db)s' % POSTGRES, echo=True)


engine.execute("CREATE SCHEMA IF NOT EXISTS "+schema+" ;")
# create schema

engine.execute("ALTER SCHEMA "+schema+" OWNER TO "+group+" ;")

engine.execute("CREATE TABLE IF NOT EXISTS "+schema+"."+table+" \
        ( id serial, \
          timestamp timestamp (2) default current_timestamp, \
          temperature integer, \
          PRIMARY KEY (id));")

engine.execute("ALTER TABLE "+schema+"."+table+" OWNER to "+group+";")
engine.execute("GRANT ALL ON ALL TABLES IN SCHEMA "+schema+" TO "+group+";")
engine.execute("GRANT ALL ON ALL SEQUENCES IN SCHEMA "+schema+" TO "+group+";")

```

This code can connect to IohHub，if it connect successful `on_connect` will print successful result and subscribe topic `/hello`，you can define topic by yourself，and when we receive message `on_message` will save data to postgresql。

```py
# mqtt connect
def on_connect(client, userdata, flags, rc):
    print("Connected with result code "+str(rc))
    client.subscribe("/hello")
    print('subscribe on /hello')


def on_message(client, userdata, msg):

    engine.execute("INSERT INTO "+str(schema)+"."+str(table) +
                   " (temperature) VALUES ("+str(msg.payload.decode())+") ",
                   echo=True)
    print('insert sueecssful')
    print(msg.topic+','+msg.payload.decode())


client = mqtt.Client()

client.username_pw_set(username, password)
client.on_connect = on_connect
client.on_message = on_message

client.connect(broker, mqtt_port, 60)
client.loop_start()
```

### 5-2. publisher.py

This file can help us publish message to topic。

Edit the **publisher.py** `broker、port、username、password` you can find in **ENSAAS_SERVICES**

- bokrer："ENSAAS_SERVICES => p-rabbitmq => externalHosts"
- port :"ENSAAS_SERVICES => p-rabbitmq => mqtt => port"
- username :"ENSAAS_SERVICES => p-rabbitmq => mqtt => username"
- password: "ENSAAS_SERVICES => p-rabbitmq => mqtt => password"

![publisher](https://tva1.sinaimg.cn/large/007S8ZIlgy1gish55bh5nj318v0u0qg3.jpg)

## 6. Kubernetes Config

### 6-1. deployment.yaml

Each user needs to adjust the variables for certification, as follows：

1. metadata >> name：py-docker-iothub-postgresql-**{user_name}**
2. student：**{user_name}**
3. image：**{docker_account}** / py-docker-iothub-postgresql：latest
4. containerPort：listen 3000
5. env >> valueFrom >> secretRef >> name：need same name in Portal-service **secret name**

![deployment](https://tva1.sinaimg.cn/large/007S8ZIlgy1giye75cn27j30my0rqjx7.jpg)

**Notice：In Portal-Services secret name**
![createSecret](https://tva1.sinaimg.cn/large/007S8ZIlly1gishp9o8q5j30qo09ignf.jpg)

### 6-2. ingress.yaml

Each user needs to adjust the variables for certification, as follows：

1. metadata >> name：py-docker-iothub-postgresql-**{user_name}**
2. host：py-docker-iothub-postgresql-**{user_name}** . **{namespace_name}** . **{cluster_name}**.en.internal
3. serviceName：need to be same name in Service.yaml **{metadata name}**
4. servicePort：same **port** in Service.yaml
   ![ingress](https://tva1.sinaimg.cn/large/007S8ZIlgy1giyec4c8e3j30mv0d2417.jpg)

### 6-3. service.yaml

Each user needs to adjust the variables for certification, as follows：

1. metadata >> name：py-docker-iothub-postgresql-**{user_name}**
2. student：**{user_name}**
3. port：same **{port}** in ingress.yaml
4. targetPort：same **{port}** in deployment.yaml **{containerPort}**
   ![service](https://tva1.sinaimg.cn/large/007S8ZIlgy1giyeeppcs0j30n00aomz1.jpg)

## 7. Docker

### 7-1. dockerfile

We first download the python:3.6 and copy this application to `/app`，and install library define in `requirements.txt`

```
FROM python:3.6-slim
WORKDIR /app
ADD . /app
RUN pip3 install -r requirements.txt
EXPOSE 3000
CMD ["python", "-u", "index.py"]
```

## 8. Deployment Application Steps

### 8-1. build Docker image

Adjust to your docker account

    $ docker build -t {docker_account / py-docker-iothub-postgresql：latest} .

### 8-2. push it to Docker Hub

    $ docker push {docker_account / py-docker-iothub-postgresql：latest}

### 8-3. create kubernetes object ( All object are in the k8s folder)

    $ kubectl apply -f k8s/

![createSecret](https://tva1.sinaimg.cn/large/007S8ZIlgy1giyeh91d6sj31du050whd.jpg)

### 8-4. Check（Pod status is running for success）

    # grep can quickly find key words
    $ kubectl get all --namespace=level2 | grep postgresql-sk-chen

![createSecret](https://tva1.sinaimg.cn/large/007S8ZIlgy1giyeikgd4jj31fg0e27dj.jpg)

### 8-5. Send message to wise-paas by MQTT

**Open two terminal first.**

    # 1. View the log of the container
    kubectl logs -f pod/{pod_name} --namespace=level2

![createSecret](https://tva1.sinaimg.cn/large/007S8ZIlgy1giyelqkqtpj312b0u04qp.jpg)

    # 2. Send message to application in WISE-PaaS
    python publisher.py

![createSecret](https://tva1.sinaimg.cn/large/007S8ZIlgy1giyenw3zuhj31ds03ggn2.jpg)

### 8-6. Check that postgresql has received data

You can use pgAdmin to check our your insert data，so you need to go to Portal-service or decode data to get your config

![copyDataVS](https://tva1.sinaimg.cn/large/007S8ZIlgy1giydu3xomxj30li0e0ad5.jpg)

go to pdAdmin(Servers => create => server)

**Connection config**

![Imgur](https://i.imgur.com/HEb9o42.png)

(Databases => Schemas => Tables => right click => View/Edit data)

![Imgur](https://i.imgur.com/Jbj8u2c.png)
