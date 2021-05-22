Data Modelling Sparkify Postgres ETL

Context

A startup called Sparkify wants to analyze the data they've been collecting on songs and user activity on their new music streaming app. The analytics team is particularly interested in understanding what songs users are listening to. Currently, they don't have an easy way to query their data, which resides in a directory of JSON logs on user activity on the app, as well as a directory with JSON metadata on the songs in their app.
The project is to create a database schema and ETL pipeline for this analysis.

Data
•	song_data - all json files are nested in subdirectories under /data/song_data. A sample of this files is:
{"num_songs": 1, 
"artist_id": "ARD7TVE1187B99BFB1", 
"artist_latitude": null,
"artist_longitude": null, 
"artist_location": "California - LA", 
"artist_name": "Casual", 
"song_id": "SOMZWCG12A8C13C480", 
"title": "I Didn't Mean To", 
"duration": 218.93179, 
"year": 0}

{"num_songs": 1, 
"artist_id": "ARD842G1187B997376", 
"artist_latitude": 43.64856, 
"artist_longitude": -79.38533, 
"artist_location": "Toronto, Ontario, Canada", 
"artist_name": "Blue Rodeo", 
"song_id": "SOHUOAP12A8AE488E9", 
"title": "Floating", 
"duration": 491.12771, 
"year": 1987}
•	Log datasets: all json files are nested in subdirectories under /data/log_data. A sample of a single row of each files is:
{"artist":"A Fine Frenzy",
"auth":"Logged In",
"firstName":"Anabelle",
"gender":"F",
"itemInSession":0,
"lastName":"Simpson",
"length":267.91138,
"level":"free",
"location":"Philadelphia-Camden-Wilmington,PA-NJ-DE-MD",
"method":"PUT",
"page":"NextSong",
"registration":1541044398796.0,
"sessionId":256,
"song":"Almost Lover (Album Version)",
"status":200,
"ts":1541377992796,
"userAgent":"\"Mozilla\/5.0 (Macintosh; Intel Mac OS X 10_9_4) AppleWebKit\/537.36 (KHTML, like Gecko) Chrome\/36.0.1985.125 Safari\/537.36\"",
"userId":"69"}

Database Schema

The schema used for this exercise is the Star Schema: There is one main fact table containing all the measures associated to each event (user song plays), and 4 dimentional tables, each with a primary key that is being referenced from the fact table.
On why to use a relational database for this case:
•	The data types are structured (we know before-hand the sctructure of the jsons we need to analyze, and where and how to extract and transform each field)
•	The amount of data we need to analyze is not big enough to require big data related solutions.
•	Ability to use SQL that is more than enough for this kind of analysis
•	Data needed to answer business questions can be modeled using simple ERD models
•	We need to use JOINS for this scenario

Fact Table

songplays - records in log data associated with song plays i.e. records with page NextSong
•	songplay_id (SERIAL) PRIMARY KEY: ID of each user song play
•	start_time (TIMESTAMP): Timestamp of beggining of user activity
•	user_id (INT) : ID of user
•	level (TEXT): User level {free | paid}
•	song_id (TEXT): ID of Song played
•	artist_id (TEXT): ID of Artist of the song played
•	session_id (INT): ID of the user Session
•	location (TEXT): User location
•	user_agent (TEXT): Agent used by user to access Sparkify platform

Dimension Tables

users - users in the app
•	user_id (INT) PRIMARY KEY: ID of user
•	first_name (TEXT): Name of user
•	last_name (TEXT): Last Name of user
•	gender (TEXT): Gender of user {M | F}
•	level (TEXT): User level {free | paid}

songs - songs in music database

•	song_id (TEXT) PRIMARY KEY: ID of Song
•	title (TEXT): Title of Song
•	artist_id (TEXT): ID of song Artist
•	year (INT): Year of song release
•	duration (FLOAT): Song duration in milliseconds

artists - artists in music database

•	artist_id (TEXT) PRIMARY KEY: ID of Artist
•	name (TEXT) NOT NULL: Name of Artist
•	location (TEXT): Name of Artist city
•	lattitude (FLOAT): Lattitude location of artist
•	longitude (FLOAT): Longitude location of artist

time - timestamps of records in songplays broken down into specific units

•	start_time (TIMESTAMP) PRIMARY KEY: Timestamp of row
•	hour (INT): Hour associated to start_time
•	day (INT): Day associated to start_time
•	week (INT): Week of year associated to start_time
•	month (INT): Month associated to start_time
•	year (INT): Year associated to start_time
•	weekday (TEXT): Name of week day associated to start_time

Project structure

Files used on the project:

1.	data folder contains song and log dataset, where all needed jsons reside.
2.	sql_queries.py contains all your sql queries, and is imported into the files bellow.
3.	create_tables.py drops and creates tables. You run this file to reset your tables before each time you run your ETL scripts.
4.	test.ipynb displays the first few rows of each table to let you check your database.
5.	etl.ipynb reads and processes a single file from song_data and log_data and loads the data into your tables.
6.	etl.py reads and processes files from song_data and log_data and loads them into your tables.
7.	README.md current file, provides discussion on my project.


Break down of steps followed

1  DROP, CREATE and INSERT query statements in sql_queries.py
2  Run in console – terminal

![](image/run_createtable.png)

3 Used test.ipynb Jupyter Notebook to interactively verify that all tables were created correctly.
4 Followed the instructions and completed etl.ipynb Notebook to create the blueprint of the pipeline to process and insert all data into the tables.
5 Once verified that base steps were correct by checking with test.ipynb, filled in etl.py program.
6 Run etl in console:
![](image/run_etl.png)
7 Verify results using test.ipynb notebook

ETL pipeline (etl.py)

1.	We start our program by connecting to the sparkify database.
2.	We walk through the tree files under /data/song_data, and for each json file encountered we use the function called process_song_file :
a. We load the file as a dataframe using a pandas function "read_json()".
b. For each row in the dataframe we select the fields we are interested in:
 ```
 song_data = [song_id, title, artist_id, year, duration]
 ```
 ```
  artist_data = [artist_id, artist_name, artist_location, artist_longitude, artist_latitude]
 ```
c. We insert this data into their respective databases.
3.	We walk through the tree files under /data/log_date, and for each json file encountered we use the function called process_log_file
a. We load our data as a dataframe same way as with songs data.
b. We select rows where page = 'NextSong' only
c. We convert ts column where we have our start_time as timestamp in millisencs to datetime format. We obtain the parameters we need from this date (day, hour, week, etc), and insert everything into our time dimentional table.
d. We store user data into our user table

e. Finally we lookup song and artist id from their tables by song name, artist name and song duration that we have on our song play data. The query used is the following: song_select = (""" SELECT song_id, artists.artist_id FROM songs JOIN artists ON songs.artist_id = artists.artist_id WHERE songs.title = %s AND artists.name = %s AND songs.duration = %s """)
f. The last step is inserting everything we need into our songplay fact table.
