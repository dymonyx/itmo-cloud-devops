FROM python:3.8-slim

WORKDIR /app

COPY /devops-sample-django-app .

COPY uwsgi.ini .

RUN apt-get update && \
    apt-get install -y libpq-dev libjpeg-dev zlib1g-dev postgresql postgresql-contrib gcc python3-dev build-essential libpcre3-dev&& \
    apt-get clean


RUN pip install --no-cache-dir -r requirements.txt


RUN pip install uwsgi

EXPOSE 8181

CMD ["uwsgi", "--ini", "uwsgi.ini"]


