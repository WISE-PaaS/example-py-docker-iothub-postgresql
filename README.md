# Example-python-Docker-Iothub-Postgresql

This is WIES-PaaS Iothub example-code include the sso and rabbitmq service，and we use the Docker package this file。

[IotHub](https://advantech.wistia.com/medias/up3q2vxvn3)

[SSO](https://advantech.wistia.com/medias/vay5uug5q6)

## Quick Start

#### Environment prepare

#### python3(need include pip3)

[python3](https://www.python.org/downloads/)

#### cf-cli

[cf-cli](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html)

Use to push application to WISE-PaaS，if you want to know more you can see this video

#### docker

[docker](https://www.docker.com/)

Use to packaged our application

#### Postgrsql

You can download pgAdmin so you can see the result in WISE-PaaS Postgresql servince instance

[Postgresql](https://www.postgresql.org/)

python3 package(those library can run this application in local):

    #mqtt
    pip3 install paho-mqtt
    #python-backend
    pip3 install Flask

    #postgresql library
    pip3 install sqlalchemy
    pip3 install psycopg2

#### Download this file

    git clone https://github.com/WISE-PaaS/example-py-docker-iothub-postgresql.git

#### Login to WISE-PaaS

![Imgur](https://i.imgur.com/JNJmxFy.png)

    #cf login -skip-ssl-validation -a {api.domain_name}  -u "account" -p "password"

    cf login –skip-ssl-validation -a api.wise-paas.io -u xxxxx@advtech.com.tw -p xxxxxx

    #check the cf status
    cf target

## Application Introduce

#### Dockerfile

We first download the python:3.6 and copy this application to `/app`，and install library define in `requirements.txt`

```
FROM python:3.6-slim
WORKDIR /app
ADD . /app
RUN pip3 install -r requirements.txt

#Use in local
# EXPOSE 3000
# CMD ["python", "hello.py"]
```

#### index.py

Simply backend appliaction。

```py
app = Flask(__name__)

# port from cloud environment variable or localhost:3000
port = int(os.getenv("PORT", 3000))


@app.route('/', methods=['GET'])
def root():

    if(port == 3000):
        return 'hello world! i am in the local'
    elif(port == int(os.getenv("PORT"))):
        return render_template('index.html')
```

This is the postgresql connect config code，`vcap_services` can get the application environment in WISE-PaaS，you need to attention，and the service_name it need to be same name in WISE-PaaS postgresql service name。


![Imgur](https://i.imgur.com/6777rmg.png)


```py

#need to be same name in WISE-PaaS service name
IOTHUB_SERVICE_NAME = 'p-rabbitmq'
DB_SERVICE_NAME = 'postgresql-innoworks'

# Get the environment variables
vcap_services = os.getenv('VCAP_SERVICES')
vcap_services_js = json.loads(vcap_services)

# --- MQTT(rabbitmq) ---
credentials = vcap_services_js[IOTHUB_SERVICE_NAME][0]['credentials']
mqtt_credential = credentials['protocols']['mqtt']

broker = mqtt_credential['host']
username = mqtt_credential['username'].strip()
password = mqtt_credential['password'].strip()
mqtt_port = mqtt_credential['port']

# --- Postgresql ---
credentials = vcap_services_js[DB_SERVICE_NAME][0]['credentials']
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


**Notice:You can add service instance by yourself**

![Imgur](https://i.imgur.com/ajqSsn1.png)

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

#### mainfest config

Open **manifest.yml** and editor the **application name**，because the appication can't duplicate in same domain name。

Check the service instance name same as WISE-PaaS

![Imgur](https://i.imgur.com/4eynKmE.png)

![Imgur](https://i.imgur.com/VVMcYO8.png)

## SSO(Single Sign On)

This is the [sso](https://advantech.wistia.com/medias/vay5uug5q6) applicaition，open **`templates/index.html`** and editor the `ssoUrl` to your application name，

If you don't want it，you can ignore it。
  
 #change this **`python-demo-try`** to your **application name**
var ssoUrl = myUrl.replace('python-demo-try', 'portal-sso');

## Build Dokcer image

Build image

    docker build -t {image} .
    docker build -t example-python-docker .

## Tag image to a docker hub  

First we need to create a repository on Docker Hub and copy it name 

[Docker Hub](https://hub.docker.com/)

![Imgur](https://i.imgur.com/SxiLcOH.png)

    #docker login to the docker hub
    docker login

    #docker tag {image name} {your account/dockerhub-resp name}
    docker tag example-py-docker WISE-PaaS/example-py-docker

Push it to Docker Hub
  
    #docker push {your account/dockerhub-resp name}
    docker push WISE-PaaS/example-py-docker

Push application and no start it because we need to bind the postgresql service first。

    #cf push --docker-image{WISE-PaaS/DOCKERHUB-RESP name}
    cf push --docker-image WISE-PaaS/eample-py-docker --no-start

Bind the postgresql application service，we already bind rabbitmq in `manifest.yml`，if you use the Linux or Mac，the bind group way is different，use `cf bs` can see it。

Notice:The `groupfamily` we define in `index.py` must be the same

    #cf bs {application name} {service instance name} -c {group}
    cf bs pythonpostgresql postgresql -c '{\"group\":\"groupfamily\"}'

Get the application environment the save it to `env.json`

    #get the application environment
    cf env python-demo-try > env.json

#### publisher.py

This file can help us publishW message to topic。

Edit the **publisher.py** `broker、port、username、password` you can find in env.json

- bokrer:"VCAP_SERVICES => p-rabbitmq => externalHosts"
- port :"VCAP_SERVICES => p-rabbitmq => mqtt => port"
- username :"VCAP_SERVICES => p-rabbitmq => mqtt => username"
- password: "VCAP_SERVICES => p-rabbitmq => mqtt => password"

open two terminal
  
 Listen the console
  
    #cf logs {application name}
    cf logs python-demo-try

send message to application in WISE-PaaS

    python publisher.py

![Imgur](https://i.imgur.com/9HEJ9OF.png)



**result**

You can use pgAdmin to chech our your insert data，so you need to go to WISE-PaaS to get your config

![Imgur](https://i.imgur.com/RciwrZq.png)

go to pdAdmin(Servers => create => server)

### connection config

![Imgur](https://i.imgur.com/HEb9o42.png)

(Databases => Schemas => Tables => right click => View/Edit data)

![Imgur](https://i.imgur.com/Jbj8u2c.png)
