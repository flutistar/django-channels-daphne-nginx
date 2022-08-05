### django-channels-daphne-nginx
Deploy Django(+Channels + Celery + Django REST Framework) project on Ubuntu 20.04 with NGINX

## 1. Install asdf (version management)
[asdf](https://github.com/asdf-vm/asdf) is a CLI tool that can manage multiple language runtime versions on a per-project basis.

- First we need to run `sudo apt update`
- Run `git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.8.0` to install asdf
- Add the below command into the ~/.bashrc
   ```
   . $HOME/.asdf/asdf.sh
   . $HOME/.asdf/completions/asdf.bash
   ```
- Restart the bash
- Run `asdf reshim`

## 2. Add Python plugin into 'asdf'
 * Run `asdf plugin add python`
 * Run `asdf install python <version>` e.g. `asdf install python 3.7.6`
 
 If python plugin doesn't install properly, you can check this url
 https://github.com/pyenv/pyenv/wiki/Common-build-problems
 
 
 * Check if python 3.7.6 is installed - `python --version`
## 3. Download the project from the github and Run it as development mode
 * `git clone <git_url>`
 * Navigate into the project dir.
 * Create Virtual Environment and activate the virtualenv
   `python -m venv venv`
   `source venv/bin/activate`
 * Install the dependencies from 'requirements.txt'
  `pip install -r requirements.txt`
 * Migrate the database and run the project to see if the dependencies are installed successfully.
  `python manage.py migrate`
  `python manage.py runserver`
  
 
 ## 4. Install guicorn and run the project by using gunicorn
  * Run `pip install gunicorn`
  * Run `sudo nano /etc/systemd/system/gunicorn.service)
  * Write the below code into the gunicorn.service file
    ```
    [Unit]
       Description=gunicorn daemon
       After=network.target

    [Service]
       User=ubuntu
       Group=www-data
       WorkingDirectory=/home/ubuntu/<project_dir>
       ExecStart=/home/ubuntu/<project_dir>/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/ubuntu/<project_dir>/gunicorn.sock <project_name>.wsgi:application

    [Install]
       WantedBy=multi-user.target

    ```
   * `sudo systemctl daemon-reload`
   * `sudo systemctl start gunicorn.service`
   * `sudo systemctl enable gunicorn.service`
   
   Check if the `gunicorn.sock` file exists in the project directory
 ## 5. Install `daphne` and run the project with `daphne` server
   * `pip install daphne`
   * `sudo nano /etc/systemd/system/daphne.service`
   * Write the below code into the `daphne.service`
   
   ```
   [Unit]
     Description=Daphne Service
     After=network.target

   [Service]
     Type=simple
     User=ubuntu
     WorkingDirectory=/home/ubuntu/<project_dir>
     ExecStart=/home/ubuntu/<project_dir>/venv/bin/daphne -u /home/ubuntu/<project_dir>/daphne.sock <project_name>.asgi:application

   [Install]
     WantedBy=multi-user.target
   ```
   * `sudo systemctl daemon-reload`
   * `sudo systemctl start daphne.service`
   * `sudo systemctl enable daphne.service`
   
   Check if the `daphne.sock` file exists in the project directory
  
  ## 6. Install `supervisor` and `redis` for Django-Celery
  
   * `sudo apt update`
   * `sudo apt install redis-server`
   * Check if the `redis-servier is installed`
      `redis-cli ping`
   * `sudo apt intall suervisor`
   * Add 2 `supervisor` configuration files into `/etc/supervisor/conf.d/`
   (It would be needed to create supervisor dir)
   * `sudo nano <celery_name>.conf`
   * Add the following
     ```
      [program:backendcelery]
      ; Set full path to celery program if using virtualenv
      command=/home/ubuntu/<project_dir>/venv/bin/celery worker -A backend --loglevel=INFO
      ; The directory to your Django project
      directory=/home/ubuntu/<project_dir>
      ; If supervisord is run as the root user, switch users to this UNIX user account before doing any processing.
      user=ubuntu
      ; Supervisor will start as many instances of this program as named by numprocs
      numprocs=1
      ; Put process stdout output in this file
      stdout_logfile=/var/log/celery/<log_file_name>.log
      ; Put process stderr output in this file
      stderr_logfile=/var/log/celery/<log_file_name>.log
      ; If true, this program will start automatically when supervisord is started
      autostart=true
      ; May be one of false, unexpected, or true. If false, the process will never be autorestarted. If unexpected, the process will be restart when the program
      ; exits with an exit code that is not one of the exit codes associated with this process’ configuration (see exitcodes). If true, the process will be
      ; unconditionally restarted when it exits, without regard to its exit code.
      autorestart=true
      ; The total number of seconds which the program needs to stay running after a startup to consider the start successful.
      startsecs=10
      ; Need to wait for currently executing tasks to finish at shutdown. Increase this if you have very long running tasks.
      stopwaitsecs = 600
      ; When resorting to send SIGKILL to the program to terminate it send SIGKILL to its whole process group instead, taking care of its children as well.
      killasgroup=true
      ; if your broker is supervised, set its priority higher so it starts first
      priority=998
     ```
   * `sudo nano <celery_beat_name>.conf`
   * Add the following
     ```
      [program:backendcelerybeat]
      ; Set full path to celery program if using virtualenv
      command=/home/ubuntu/<project_dir>/venv/bin/celery beat -A backend --loglevel=INFO
      ; The directory to your Django project
      directory=/home/ubuntu/<project_dir>
      ; If supervisord is run as the root user, switch users to this UNIX user account before doing any processing.
      user=ubuntu
      ; Supervisor will start as many instances of this program as named by numprocs
      numprocs=1
      ; Put process stdout output in this file
      stdout_logfile=/var/log/celery/<log_file_name>.log
      ; Put process stderr output in this file
      stderr_logfile=/var/log/celery/<log_file_name>.log
      ; If true, this program will start automatically when supervisord is started
      autostart=true
      ; May be one of false, unexpected, or true. If false, the process will never be autorestarted. If unexpected, the process will be restart when the program
      ; exits with an exit code that is not one of the exit codes associated with this process’ configuration (see exitcodes). If true, the process will be
      ; unconditionally restarted when it exits, without regard to its exit code.
      autorestart=true
      ; The total number of seconds which the program needs to stay running after a startup to consider the start successful.
      startsecs=10
      ; if your broker is supervised, set its priority higher so it starts first
      priority=999
     ```
   * Create the log files
     ```
     touch /var/log/celery/backend_worker.log
     touch /var/log/celery/backend_beat.log
     ```
   * `sudo supervisorctl reread`
   * `sudo supervisorctl update`
   * Run the following commands to stop, start, and/or check the status of the celery program:
     `sudo supervisorctl stop <celery_name>`
     `sudo supervisorctl start <celery_name>`
     `sudo supervisorctl status <celery_name>`
 
 ## 7. NGINX configuration
   * `sudo nano /etc/nginx/sites-available`
   * Add the following
     ```
     server {
         listen 80;
         server_name ;

         location = /favicon.ico { access_log off; log_not_found off; }
         location /static/ {
             root /home/ubuntu;
         }

         location / {
             include proxy_params;
             proxy_pass http://unix:/home/ubuntu/<project_dir>/daphne.sock;
         }
         #/notification/ is redirecting the requests into the daphne.sock
         location /notification/ {
             proxy_pass http://unix:/home/ubuntu/<project_dir>/daphne.sock;
             proxy_http_version 1.1;
             proxy_set_header Upgrade $http_upgrade;
             proxy_set_header Connection "upgrade";
             proxy_redirect off;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header X-Forwarded-Host $server_name;
         }
     }
     ```
