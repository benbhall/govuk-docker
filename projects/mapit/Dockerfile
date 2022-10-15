FROM python:3.10.7-buster

RUN apt-get update -qq && apt-get install -y gettext \
        postgresql-client libgdal-dev libgeos-dev musl-dev ruby-dev

RUN git clone https://github.com/alphagov/mapit.git

WORKDIR /mapit

RUN pip install shapely six
RUN pip install --upgrade pip wheel setuptools
RUN pip install -r requirements.txt
