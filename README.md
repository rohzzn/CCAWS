# Flask Registration and Login Web Application on AWS EC2

This repository contains a Flask web application deployed on an AWS EC2 instance. The application allows users to register, log in, upload a text file, display the word count of the uploaded file, and download the file. User information is stored in a SQLite3 database.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Setup and Installation](#setup-and-installation)
  - [1. Launch an EC2 Instance](#1-launch-an-ec2-instance)
  - [2. Connect to Your EC2 Instance](#2-connect-to-your-ec2-instance)
  - [3. Install Required Software](#3-install-required-software)
  - [4. Set Up the Flask Application](#4-set-up-the-flask-application)
  - [5. Configure Apache to Serve the Flask App](#5-configure-apache-to-serve-the-flask-app)
  - [6. Access the Web Application](#6-access-the-web-application)
- [Usage](#usage)
  - [Registration](#registration)
  - [Login](#login)
- [Extra Credit Features](#extra-credit-features)
- [Screenshots](#screenshots)
- [Security Considerations](#security-considerations)
- [References](#references)
- [License](#license)

## Overview

This project demonstrates how to deploy a Flask web application on an AWS EC2 instance, configure it with Apache and mod_wsgi, and connect it to a SQLite3 database. It includes user registration and login functionalities, as well as file upload and word count display features.

## Features

- User Registration
  - Username
  - Password
  - First Name
  - Last Name
  - Email
- User Login
- SQLite3 Database Integration
- File Upload (Text Files)
- Word Count Display of Uploaded File
- Download Link for Uploaded File
- Display of User Information upon Relogin

## Prerequisites

- AWS Account
- Basic knowledge of Linux command line
- GitHub account (for code repository)

## Setup and Installation

### 1. Launch an EC2 Instance

#### a. Sign in to AWS Management Console

- Navigate to the [AWS Management Console](https://aws.amazon.com/console/).
- Sign in with your AWS credentials.

#### b. Navigate to EC2 Dashboard

- Click on **Services** and select **EC2** under **Compute**.

#### c. Launch an Instance

- Click on **Launch Instance**.

#### d. Choose an Amazon Machine Image (AMI)

- Search for **Ubuntu Server 22.04 LTS** (latest LTS version).
- Select the **Ubuntu Server 22.04 LTS (HVM), SSD Volume Type** AMI.

#### e. Choose an Instance Type

- Select **t2.micro** (or **t3.micro** if t2.micro is unavailable).

#### f. Configure Instance Details

- Keep the default settings.
- Ensure **Auto-assign Public IP** is enabled.

#### g. Add Storage

- Use the default storage size (8 GiB).

#### h. Add Tags (Optional)

- Add a tag to identify your instance (e.g., **Name**: FlaskAppInstance).

#### i. Configure Security Group

- Create a new security group with the following inbound rules:

  | Type        | Protocol | Port Range | Source          | Description       |
  |-------------|----------|------------|-----------------|-------------------|
  | SSH         | TCP      | 22         | Your IP         | SSH access        |
  | HTTP        | TCP      | 80         | 0.0.0.0/0       | Web server access |

#### j. Review and Launch

- Review your configuration and click **Launch**.
- Select an existing key pair or create a new one.
- Download the key pair and keep it secure.

### 2. Connect to Your EC2 Instance

#### a. Set Permissions for Key Pair File

```bash
chmod 400 your-key-pair.pem
```

#### b. Connect via SSH

```bash
ssh -i "your-key-pair.pem" ubuntu@ec2-your-instance-public-dns.amazonaws.com
```

### 3. Install Required Software

#### a. Update Packages

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

#### b. Install Apache and mod_wsgi

```bash
sudo apt-get install apache2 -y
sudo apt-get install libapache2-mod-wsgi-py3 -y
```

#### c. Install Python 3, Pip, and Flask

```bash
sudo apt-get install python3-pip -y
sudo pip3 install flask
```

#### d. Install SQLite3

```bash
sudo apt-get install sqlite3 -y
```

#### e. Set Permissions

```bash
sudo chmod 755 /home/ubuntu
```

### 4. Set Up the Flask Application

#### a. Create Application Directory

```bash
cd /var/www
sudo mkdir flaskapp
sudo chown -R ubuntu:ubuntu flaskapp
cd flaskapp
```

#### b. Initialize Git Repository

```bash
git init
```

#### c. Create Virtual Environment (Optional but Recommended)

```bash
sudo apt-get install python3-venv -y
python3 -m venv venv
source venv/bin/activate
```

#### d. Create `app.py`

Create `app.py` with the following content:

```python
from flask import Flask, render_template, request, redirect, url_for, send_from_directory
import sqlite3
import os
from werkzeug.utils import secure_filename

app = Flask(__name__)

# Configuration for file uploads
UPLOAD_FOLDER = os.path.join(app.root_path, 'uploads')
ALLOWED_EXTENSIONS = {'txt'}

app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER

# Ensure the upload folder exists
if not os.path.exists(UPLOAD_FOLDER):
    os.makedirs(UPLOAD_FOLDER)

# Initialize the database and create the users table if it doesn't exist
def init_db():
    conn = sqlite3.connect('users.db')
    c = conn.cursor()
    # Create table with additional fields for filename and word count
    c.execute('''CREATE TABLE IF NOT EXISTS users (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    username TEXT NOT NULL,
                    password TEXT NOT NULL,
                    firstname TEXT NOT NULL,
                    lastname TEXT NOT NULL,
                    email TEXT NOT NULL,
                    filename TEXT,
                    word_count INTEGER
                )''')
    conn.commit()
    conn.close()

init_db()

# Function to check allowed file extensions
def allowed_file(filename):
    return '.' in filename and \
           filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

# Route for the registration page
@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        # Get form data
        username = request.form['username']
        password = request.form['password']
        firstname = request.form['firstname']
        lastname = request.form['lastname']
        email = request.form['email']

        # Handle file upload
        file = request.files['file']
        filename = None
        word_count = 0

        if file and allowed_file(file.filename):
            filename = secure_filename(file.filename)
            file_path = os.path.join(app.config['UPLOAD_FOLDER'], filename)
            file.save(file_path)

            # Count words in the file
            with open(file_path, 'r') as f:
                contents = f.read()
                word_count = len(contents.split())

        # Save user data to the database
        conn = sqlite3.connect('users.db')
        c = conn.cursor()
        c.execute("""INSERT INTO users (username, password, firstname, lastname, email, filename, word_count)
                     VALUES (?, ?, ?, ?, ?, ?, ?)""",
                  (username, password, firstname, lastname, email, filename, word_count))
        conn.commit()
        conn.close()

        return redirect(url_for('profile', username=username))
    else:
        return render_template('register.html')

# Route for the user profile page
@app.route('/profile/<username>')
def profile(username):
    conn = sqlite3.connect('users.db')
    c = conn.cursor()
    c.execute("SELECT * FROM users WHERE username=?", (username,))
    user = c.fetchone()
    conn.close()

    return render_template('profile.html', user=user)

# Route for the login page
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        # Get login credentials
        username = request.form['username']
        password = request.form['password']

        # Check credentials against the database
        conn = sqlite3.connect('users.db')
        c = conn.cursor()
        c.execute("SELECT * FROM users WHERE username=? AND password=?", (username, password))
        user = c.fetchone()
        conn.close()

        if user:
            return redirect(url_for('profile', username=username))
        else:
            return "Invalid credentials"
    else:
        return render_template('login.html')

# Route to serve uploaded files
@app.route('/uploads/<filename>')
def uploaded_file(filename):
    return send_from_directory(app.config['UPLOAD_FOLDER'], filename)

if __name__ == '__main__':
    app.run(debug=True)
```

#### e. Create Templates Directory and HTML Files

Create a `templates` directory and add the following HTML files.

##### `templates/register.html`

```html
<!DOCTYPE html>
<html>
<head>
    <title>Registration Page</title>
</head>
<body>
<h1>Registration Page</h1>
<form method="POST" action="/register" enctype="multipart/form-data">
    Username: <input type="text" name="username" required><br>
    Password: <input type="password" name="password" required><br>
    First Name: <input type="text" name="firstname" required><br>
    Last Name: <input type="text" name="lastname" required><br>
    Email: <input type="email" name="email" required><br>
    Upload File: <input type="file" name="file"><br>
    <input type="submit" value="Register">
</form>
</body>
</html>
```

##### `templates/login.html`

```html
<!DOCTYPE html>
<html>
<head>
    <title>Login Page</title>
</head>
<body>
<h1>Login Page</h1>
<form method="POST" action="/login">
    Username: <input type="text" name="username" required><br>
    Password: <input type="password" name="password" required><br>
    <input type="submit" value="Login">
</form>
</body>
</html>
```

##### `templates/profile.html`

```html
<!DOCTYPE html>
<html>
<head>
    <title>User Profile</title>
</head>
<body>
<h1>User Profile</h1>
<p>Username: {{ user[1] }}</p>
<p>First Name: {{ user[3] }}</p>
<p>Last Name: {{ user[4] }}</p>
<p>Email: {{ user[5] }}</p>

{% if user[7] %}
    <p>Word Count in Uploaded File: {{ user[7] }}</p>
{% endif %}

{% if user[6] %}
    <a href="{{ url_for('uploaded_file', filename=user[6]) }}">Download Uploaded File</a>
{% endif %}

</body>
</html>
```

#### f. Create Uploads Directory

```bash
mkdir uploads
chmod 755 uploads
```

### 5. Configure Apache to Serve the Flask App

#### a. Create a WSGI File

Create `/var/www/flaskapp/flaskapp.wsgi` with the following content:

```python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, '/var/www/flaskapp')

from app import app as application
```

#### b. Configure Apache Virtual Host

Create a new Apache configuration file `/etc/apache2/sites-available/flaskapp.conf`:

```apache
<VirtualHost *:80>
    ServerName ec2-your-instance-public-dns.amazonaws.com

    WSGIDaemonProcess flaskapp threads=5
    WSGIScriptAlias / /var/www/flaskapp/flaskapp.wsgi

    <Directory /var/www/flaskapp>
        Require all granted
    </Directory>

    Alias /static /var/www/flaskapp/static
    <Directory /var/www/flaskapp/static/>
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Replace `ec2-your-instance-public-dns.amazonaws.com` with your instance's public DNS.

#### c. Enable the Site and Restart Apache

```bash
sudo a2ensite flaskapp
sudo service apache2 restart
```

### 6. Access the Web Application

- Navigate to `http://ec2-your-instance-public-dns.amazonaws.com/register` in your web browser.
- Replace `ec2-your-instance-public-dns.amazonaws.com` with your instance's public DNS.

## Usage

### Registration

- Fill out the registration form.
- Upload a `.txt` file (e.g., `Limerick-1.txt`).
- Submit the form.
- You will be redirected to your profile page displaying your information.

### Login

- Navigate to `/login`.
- Enter your username and password.
- Upon successful login, you will be redirected to your profile page.

## Extra Credit Features

- Upload a text file during registration.
- Display the word count of the uploaded file on the profile page.
- Provide a download link for the uploaded file.
- Display all the information upon relogin.


## Security Considerations

- **Password Storage**: Passwords are stored in plain text in the database. In a production environment, always hash passwords using a secure algorithm like bcrypt.
- **File Uploads**: Only `.txt` files are allowed. In a production environment, implement additional checks to prevent malicious file uploads.
- **Database Security**: SQLite3 is used for simplicity. For production, consider using a more robust database system.

## References

- [Running a Flask app on AWS EC2](https://www.datasciencebytes.com/bytes/2015/02/24/running-a-flask-app-on-aws-ec2/)
- [Flask Documentation](https://flask.palletsprojects.com/)
- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/index.html)
- [SQLite3 Documentation](https://www.sqlite.org/docs.html)
- [GitHub: Connecting to GitHub with SSH](https://docs.github.com/authentication/connecting-to-github-with-ssh)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
