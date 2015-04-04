Back in October 2014 I attended Launch Hack, a Hackathon organised by the fantastic MLH at Bloomburg's European headquarters. My two friends ([Jordi](https://twitter.com/Jordisk) and [Tom](http://www.tomperegrine.me/)) and I decided to build a model rocket stability calculator and part of this hack involved having database of rockets. Now, unfortunately, we struggled to find a way of easily setting up a database in Python for deployment on Heroku.  However, several months later I have finally learnt how to setup and deploy a database and use it in a python app on heroku but it took several tutorials and stack overflow questions (which will be linked below), so I have decided to create this post to serve as a brief guide on how to setup and deploy a Python/SQL app on Heroku.

**please note - all commands are for a bash POSIX, they may not work in the windows command prompt and the python version I am using for this tutorial is Python 2.7**



To start off we are going to need the pip package manager for python if you don't have it go ahead and grab it here ().
Next, we are going to create a virtualenv to stop any packages we install from interfering from our global installation of python. To install virtualenv run:

`pip install virtualenv`

Then we are going to make a folder for our project and move into it (feel free to give it a shorter name):

`mkdir heroku-python-db-test`

`cd heroku-python-db-test`

Now we are inside our folder we are going to create a virtualenv, so run:

`virtualenv venv`

This will create a folder called venv in our current directory called venv and it will contain our python installation for this project. Now to use the python installation in venv run:

`source venv/bin/activate`

You should now see `(venv)` at the beginning of your prompt. Now, for our little app we are going to install some packages, please run:

`pip install flask`

`pip install peewee`

`pip install psycopg2`

`pip install gunicorn`

What we use these packages will become apparant later on but I will give a quick overview now. Flask is a python microframework and it's what allows us to use python to build web applications. Peewee is ORM which basically means instead of having to write SQL to interact with databases with Python objects - this is great because not only does it mean we can work with databases without having to learn at new language (although I would highly recomend learning SQL) but we can also can change database types which is useful when working heroku as we will use a postgresql database when we deploy but on our local machine it is easier to use SQLite 3. Psycopg2 is a package than allows peewee to connect to Postgresql databases, we don't need to install a similar package for SQLite as python already includes one.  Finally, gunicorn is a python web server (flask already includes it's own web server but we'll use this one when we deploy).

Our simple application will be really simple and pointless. We'll have one table called Names with 2 columns id and name. Then our application will have two routes (routes basically means url) on where we will see all the names and one we can a name. Our code will be split into two files: app.py and models.py, to get started create models.py in our heroku-python-db-test directory and type:

```python
from peewee import *
import os
import urlparse

if 'HEROKU' in os.environ:
    urlparse.uses_netloc.append('postgres')
    url = urlparse.urlparse(os.environ['DATABASE_URL'])

    # for your config
    database = {
        'engine': 'peewee.PostgresqlDatabase',
        'name': url.path[1:],
        'user': url.username,
        'password': url.password,
        'host': url.hostname,
        'port': url.port,
    }
    db = PostgresqlDatabase(database["name"],user=database["user"],password=database["password"],host=database["host"],port=database["port"])
else:
    db = SqliteDatabase("my_database.db")
    
class Names(Model):
	name = CharField()

	class Meta:
		database = db

def initialise():
	db.connect()
	db.create_tables([Names])
	db.close()
```

We start by importing any modules and packages we will need (peewee, os and urlparse) then we check if our code is running on heroku by checking if the HEROKU environment variable is present. If we are on heroku we get the DATABASE_URL environment variable and parse it into a dictionary called database. Then we create a variable called db which stores a PostgresqlDatabase and uses the parameters we parse into dictionary database. Otherwise if we are not on heroku we create a sqlite database called my_database.db.

Next we define a class called Names with inheriets from the class Model (which is imported from peewee) and this will represent our Names table. Inside our class we create a variable called name which is a CharField (again charfield is imported from peewee). This will represent the name column in our database. Inside the names class we create a class called Meta. The Meta class will contain extra settings about a table. In our case we tell it the database we are using is the one in the db variable we created earlier. 

Finally we declared a procedure called initialise which connects to the database, creates the Names table and the closes the connection.

Next also in our directory we will create app.py and type:

```python
from flask import Flask, g, redirect, url_for
import models

app = Flask(__name__)
app.config['DEBUG'] = True

@app.before_request
def before_request():
    """Connect to database before each request"""
    models.db.connect()

@app.after_request
def after_request(response):
    """Close the database connection after each response"""
    models.db.close()
    return response

@app.route("/")
def index():
	query = models.Names.select()
	names = ""
	for name in query:
		names += " {}".format(name.name)
	return "hello " + names

@app.route("/add/<name>")
def add(name):
	add_name = models.Names.create(name=name)
	add_name.save()
	return redirect(url_for("index"))

try:
	models.initialise()
except:
	None
```

We start by importing what we need from flask and importing our models file.

Then we create new flask app and set debug to True. 

Next we write two functions, one to execute before a request and one to execute after. So before each request we are going to open a connection the database and after each request we close the connection to the database and return what ever is passed in. Next


