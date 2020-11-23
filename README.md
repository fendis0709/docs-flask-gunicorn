# Create Flask Project using Gunicorn

## Create Flask App

1. Create new directory project
    ```bash
    mkdir /var/www/flask-app
    ```
2. Change location to your new created directory project
    ```bash
    cd /var/www/flask-app
    ```
3. Create new Virtualenv for specific python version
	```bash
	virtualenv venv --python=python3.9
	```
4. Activate your virtual environment (venv)
	```bash
	source venv/bin/activate
	```
5. Install Flask using `pip`
	```bash
	pip install flask
	```
6. Create new file `app.py` in root project directory
	```python
	from flask import Flask, request, abort, jsonify

	app = Flask(__name__)

	@app.route('/getSquare', methods=['POST'])
	def get_square():
	    if not request.json or 'number' not in request.json:
	        abort(400)
	    num = request.json['number']
	    return jsonify({'answer': num ** 2})

	if __name__ == '__main__':
	    app.run(host='127.0.0.1', port=8080, debug=True)
	```
7. Finally, your project directory should be like this
	```bash
	├── app.py
	├── venv
	│   ├── bin
	│   ├── lib
	│   ├── share
	```
    ___
    ### Testing
    1. Make sure you've been activating your virtual environment
	    ```bash
	    source venv/bin/activate
	    ```
    2. Run flask app
	    ```bash
	    python app.py
	    ```
    3. Create new request
    	```bash
		curl -i -H "Content-Type: application/json" -X POST -d '{"number":5}' http://127.0.0.1:8080/get-square
		```
		After running that command, you should get this message
		```
		HTTP/1.0 200 OK
		Content-Type: application/json
		Content-Length: 19
		Server: Werkzeug/0.16.0 Python/3.9.0
		Date: Mon, 23 Nov 2020 15:18:14 GMT  

		{ 
		    "answer": 25
		}
		```

## Install & Configuring Gunicorn

### A. Install Gunicorn

1. Install Gunicorn package
	```bash
	pip install gunicorn
	```
2. Create new file on your root project directory named `wsgi.py`
	```python
	from app import app

	if __name__ == '__main__':
	    app.run()
	```
3. Finally, your project directory should look like this
	```bash
	├── app.py
	├── venv
	│   ├── bin
	│   ├── lib
	│   ├── share
	├── wsgi.py
	```
    ___
    ### Testing
    1. Make sure you've been activating virtual environment
	    ```bash
	    source venv/bin/activate
	    ```
    2. Run flask app using Gunicorn web service
		```bash
		gunicorn --bind 127.0.0.1:8080 wsgi:app
		```
        You should get this message after running that command
        ```bash
        [2020-11-23 06:49:16 +0000] [4166] [INFO] Starting gunicorn 20.0.4
        [2020-11-23 06:49:16 +0000] [4166] [INFO] Listening at: http://127.0.0.1:8080 (4166)
        [2020-11-23 06:49:16 +0000] [4166] [INFO] Using worker: sync
        [2020-11-23 06:49:16 +0000] [4181] [INFO] Booting worker with pid: 4181
        ```
        > Alternative, you can use IP address "0.0.0.0:8080", instead of "127.0.0.1:8080". Especially in production server.

### B. Configuring Gunicorn

1. Create new file named `gunicorn.py` inside root project directory
	```python
	import multiprocessing

	workers = multiprocessing.cpu_count() * 2 + 1
	bind = 'unix:flask-app.sock'
	umask = 0o007
	reload = True

	#logging
	accesslog = '-'
	errorlog = '-'
	```
2. Create systemd file to automatically start Gunicorn service
	```bash
	sudo nano /etc/systemd/system/flask-app.service
	```
3.  Below is the content of `flask-app.service` file
	```bash
	[Unit]
	Description=Gunicorn instance to serve flask application
	After=network.target

	[Service]
	User=root
	Group=www-data
	WorkingDirectory=/var/www/flask-app/
	ExecStart=/var/www/flask-app/venv/bin/gunicorn --config gunicorn.py wsgi:app
	Environment="PATH=/home/root/flask_rest/flaskvenv/bin"
	Environment="FOO=Bar"

	[Install]
	WantedBy=multi-user.target
	```
	> You could add some configuration (DB connection, App version, etc) by using Environment="FOO=Bar". Just add some couple config there.
4. Let's start and enable the service
	```bash
	sudo systemctl start flask-app.service
	sudo systemctl enable flask-app.service
	```
5. Finally, you can check your service whether its running or not
	```bash
	flask-app.service - Gunicorn instance to serve flask application
	   Loaded: loaded (/etc/systemd/system/flask-app.service; enabled; vendor preset: enabled)
	   Active: active (running) since Mon 2020-11-23 04:55:01 UTC; 1h 4min ago
	 Main PID: 19207 (gunicorn)
	    Tasks: 11 (limit: 4647)
	   CGroup: /system.slice/flask-app.service
	   ...
	```

## Configure Virtual Host (Apache)

1. Create Apache virtual host config file
	```bash
	sudo nano /etc/apache2/sites-available/flask-app.conf
	```
2. Below is the content of `flask-app.conf`
	```bash
	<VirtualHost *:80>

	  ServerAdmin support@example.com
	  ServerName  flask-app.example.com

	  ErrorLog ${APACHE_LOG_DIR}/flask-app-error.log
	  CustomLog ${APACHE_LOG_DIR}/flask-app-access.log combined

	  <Location />
	    ProxyPass unix:/var/www/flask-app/flask-app.sock|http://127.0.0.1:8080/
	    ProxyPassReverse unix:/var/www/flask-app/flask-app.sock|http://127.0.0.1:8080/
	  </Location>

	</VirtualHost>
	```

3. Enable new Apache config
	```bash
	sudo a2ensite flask-app.conf
	```
4. Reload Apache service
	```bash
	sudo service apache2 reload
	```

## Finally, It's over. Party time.

Congratulation, you have been created new project using Flask and Gunicorn. Now, it's time to test our API.

1. Create new request using cURL
	```bash
	curl -i -H "Content-Type: application/json" -X POST -d '{"number":5}' http://flask-app.example.com
	```
2. You should get this message
	```
	HTTP/1.0 200 OK
	Content-Type: application/json
	Content-Length: 19
	Server: Werkzeug/0.16.0 Python/3.9.0
	Date: Mon, 23 Nov 2020 15:18:14 GMT  

	{ 
        "answer": 25
	}
	```

___

### Pro Tip

1. Save your pip package into text file and install them when deploy to your server
    1. Save `pip` packages into text file
	    ```bash
	    pip freeze > requirements.txt
	    ```
    3. Install`pip` packages from text file
	    ```bash
	    pip install -r requirements.txt
	    ``` 


> Reference : https://medium.com/@thishantha17/build-a-simple-python-rest-api-with-apache2-gunicorn-and-flask-on-ubuntu-18-04-c9d47639139b
