![TeamCity CodeBetter](https://img.shields.io/teamcity/codebetter/bt428.svg)
![Packagist](https://img.shields.io/packagist/l/doctrine/orm.svg)
![PyPI - Python Version](https://img.shields.io/pypi/pyversions/Django.svg)



# FullStack: Flask + React + Docker + GraphQL
[Flask](http://flask.pocoo.org/) is a very powerful framework. You can build a vast variety of systems from a very basic web application to a large platform using flask. [React](https://reactjs.org) is a very popular, easy to use & very powerful front-end development JavaScript library. [Docker](https://www.docker.com/) is an open platform for developers and system administrators to build, ship, and run distributed applications, whether on laptops, data centre VMs, or the cloud. [GraphQL](https://graphql.org/) is a query language for APIs and a runtime for fulfilling those queries with your existing data. GraphQL provides a complete and understandable description of the data in your API, gives clients the power to ask for exactly what they need and nothing more, makes it easier to evolve APIs over time, and enables powerful developer tools.

### SSH
1. Copy the SSH key to your clipboard.

If your SSH key file has a different name than the example code, modify the filename to match your current setup. When copying your key, don't add any newlines or whitespace.
```
$ pbcopy < ~/.ssh/id_rsa.pub
# Copies the contents of the id_rsa.pub file to your clipboard
```
>**Tip**: If pbcopy isn't working, you can locate the hidden .ssh folder, open the file in your favorite text editor, and copy it to your clipboard.

2. In the upper-right corner of any page, click your profile photo, then click **Settings**.

3. In the user settings sidebar, click **SSH and GPG keys**.

4. Click **New SSH key or Add SSH key.**

5. In the "Title" field, add a descriptive label for the new key. For example, if you're using a personal Mac, you might call this key "Personal MacBook Air".

6. Paste your key into the "Key" field.

7. Click **Add SSH key.**

8. If prompted, confirm your GitHub password.


```
git clone git@github.com:mattd429/flask-reactjs-docker-graphql.git
```

### DockerHub

You can also grab these containers on [dockerhub](https://hub.docker.com/r/mattd429/flask_react_docker/tags/)

![alt text](https://i.imgur.com/MUbl7u2.png)

### IDE:

It's good practice to use a IDE that works with your current environment, below I set this project up using [VSCode](https://code.visualstudio.com/).  I chose this since we are using Python's Flask microframework as our backend.  You can choose anything that works for you.

---

Below is the directory structure, you can follow this structure or create your own.

> **NOTE**: this is just an example of how to start your project structure, keep in mind this will change overtime.  Once this project is complete I will add the final stack image below and remove this note.

![alt text](https://i.imgur.com/aBqNScy.png)

- **instance** - This is located outside the app package and can hold local
    data that shouldn't be committed to version control, such as configuration secrets
    and database files.
- **app.py**: This is the entry point for the Flask web-server
- **venv**: This is to setup your virtual environment in Python, it seperates itself from the OS build.
- **requirements.txt**: Will install any Python Packages you will need for your app to run properly.
- **Dockerfile**: Dockerfile to build the docker containers.
- **docker-compose.yml**: Configuration development.
- **modules**: The directory that will contain all the modules, i.e. app, logger, etc.
- **app**: The main directory for the web server, where we build Flask.
- **logger**: Log errors and info into `output.log`.
- **client**: Frontend React Framework.


Start by creating `__init__.py` in your "app" module.
> **NOTE**: This project will be setup with Sentry(error checking), define a factory function(which creates another object).  The project will have other configurations added in the final package, but here I will add the default layout below. 
```python
import os

from Flask import flask
from sentry_sdk import configured_scope
with configure_scope() as scope:
    scope.user = {"email": "youremail@email.com"}
    
""" use raven to import Sentry """
from raven.contrib.flask import Sentry
sentry = Sentry(dsn='YOUR_DSN_HERE')

def create_app():
    app = Flask(__name__)
    sentry.init_app(app)
    ...
    return app
```
`modules/app/__init__.py`

- This will create the app object of Flask.

**[Sentry](https://sentry.io/welcome/)** is Open-source error tracking that helps developers monitor and fix crashes in real time. Iterate continuously. Boost efficiency. Improve user experience.  When you sign up for Sentry you will be provided with a dsn(data source name).  please visit the link above for more details.

Next create your `app.py` which will execute the Flask server using REST APIs calls.

```python
# Python imports
import os
import sys
# Flask
from flask import jsonify, make_response, send_from_directory
from flask_graphql import GraphQLView
from flask_cors import CORS
# Local
from modules import logger
from modules.app import create_app
from modules.app.database import db_session, init_db
from modules.app.schema import schema

app = create_app()
app.debug = True

default_query = '''
{
  Users {
    edges {
      node {
        id,
        username,
        email,
        password
        blog {
          id,
          title,
          text
        },
        role {
          id,
          name
        }
      }
    }
  }
}
'''.strip()


"""
:ROOT_PATH : Set root path
:PUBLIC_PATH : Always for joining react public index.html to ROOT_PATH
"""
ROOT_PATH = os.path.dirname(os.path.realpath(__file__))
os.environ.update({'ROOT_PATH': ROOT_PATH})
PUBLIC_PATH = os.path.join(ROOT_PATH, 'modules', 'client', 'public')

''' Set the view for Graphiql '''
view_func = GraphQLView.as_view(
    'graphql', schema=schema, graphiql=True)

""" logger object to output info and debug information """
LOG = logger.get_root_logger(os.environ.get(
    'ROOT_LOGGER', 'root'), filename=os.path.join(ROOT_PATH, 'output.log'))

PORT = os.environ.get('PORT')

app.add_url_rule('/graphql', view_func=view_func)

""" Sets routes for Flask """
@app.errorhandler(404)
def not_found(error):
    """ error handler """
    LOG.error(error)
    return make_response(jsonify({'error': 'Not Found'}), 404)

@app.route('/')
def index():
    """ serve index.html """
    return send_from_directory(PUBLIC_PATH, 'index.html')

@app.route('/login')
def login():
    """ serve index.html """
    return send_from_directory(PUBLIC_PATH, 'index.html')

@app.route('/<path:path>')
def static_proxy(path):
    """
    static folder serve
    :param path: to load all static files in `public`
    :return: the directory `public` with filename i.e. 404.html
    """
    file_name = path.split('/')[-1]
    dir_name = os.path.join(PUBLIC_PATH, '/'.join(path.split('/')[:-1]))
    return send_from_directory(dir_name, file_name)

@app.teardown_appcontext
def shutdown_session(exception=None):
    db_session.remove()

if __name__ == "__main__":
    init_db()
    CORS(app, resources={r'/graphql': {'origins': '*'}})
    LOG.info('running environment: {}'.format(os.environ.get('ENV')))
    app.config['DEBUG'] = os.environ.get('ENV') == 'development'
    app.run(host='0.0.0.0', port=int(PORT))
```    

`ROOT_PATH` actually adds "index.html" file in sys, that way we can directly access that file.  For **PORT** and **ENV** we will define those in our `docker-compose.yml` file. We have define 3 default routes, one for `index.html`, another for **error handling**, and one more to serve all static files in `dist` directory.

If you want to use logger for debugging, info, and warnings below is a basic setup of logger to the `output.log` file.

 - Add to `modules/logger/__init__.py`
 
`from . logger import *`

```python
import os
import logging


def get_root_logger(logger_name, filename=None):
    logger = logging.getLogger(logger_name)
    debug = os.environ.get('ENV', 'development') == 'development'
    logger.setLevel(logging.DEBUG if debug else logging.INFO)

    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s')

    ch = logging.StreamHandler()
    ch.setFormatter(formatter)
    logger.addHandler(ch)

    if filename:
        fh = logging.FileHandler(filename)
        fh.setFormatter(formatter)
        logger.addHandler(fh)

    return logger

def get_child_logger(root_logger, name):
    return logging.getLogger('.'.join([root_logger, name]))
```

This `logger.py` in the simpliest form will format a nice string format of your log message to your `output.log` file you will create in your root directory.

> **NOTE**: Please make sure to add all required packages to you `requirements.txt`

```
Flask==1.0.2
Flask-graphql==2.0.0
Flask-cors==3.0.6
Flask-RESTful==0.3.6
Graphql-core==2.1
Graphene[sqlalchemy]
graphql-relay
Raven[flask]
Sentry-sdk==0.3.5
SQLAlchemy==1.2.13
```

#### Setup Dockerfile:

```
FROM python:3.6
ADD . /app
WORKDIR /app
EXPOSE 4000
RUN pip install -r requirements.txt
ENTRYPOINT ["python","app.py"]
```

#### Setup docker-compose.yml

```
version: '3.5'
services:
  flask:
    build: 
      context: ./modules/app
      dockerfile: Dockerfile
    ports:
      - "4000:4000"
    volumes:
      - .:/app
    environment:
      - FLASK_ENV=development
      - PORT=4000
      - DB=mongo://mongodb:27017/dev
  react:
    build:
      context: ./modules/client
      dockerfile: Dockerfile
    volumes:
      - './modules/client:/usr/src/app'
      - '/usr/src/app/node_modules'
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - PORT=3000
```

  Now your have a bare minimum application with flask server and react client.  Run the following command below:

` $ docker-compose up --build`

Your output should look something like this:

![alt text](https://i.imgur.com/qzKUiJV.png)

Access [localhost](http://localhost:4000/) in your browser, and you will see the `index.html` loaded.
